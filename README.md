# HTPasswd operator and MTV migration permissions

Verified on luke, MTV 2.12.10, 2026-11-12. Plan: `openshift-mtv/windows-test-perms`; destination: `virt-windows`; source VM: `Windows2025` (`vm-46004`). This procedure starts an existing, administrator-prepared plan via the Kubernetes API, not the console plan editor.

## 1. Configure HTPasswd authentication

Follow [Red Hat's OpenShift 4.22 HTPasswd procedure](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/authentication_and_authorization/configuring-identity-providers#identity-provider-htpasswd-about_configuring-htpasswd-identity-provider). Red Hat recommends this only for development, not production.

As cluster administrator:

```bash
umask 077
# Install httpd-tools on RHEL, apache2-utils on Debian/Ubuntu.
# Interactive password entry avoids placing the password in shell history.
htpasswd -c -B users.htpasswd operator
oc create secret generic htpasswd -n openshift-config \
  --from-file=htpasswd=users.htpasswd
```

The cluster initially had no identity providers. Configure:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd
```

Apply this as `oc apply -f manifests/oauth.yaml`. On an existing installation, preserve other identity provider entries instead of replacing their array. Wait for `oc rollout status deployment/oauth-openshift -n openshift-authentication --timeout=180s` before testing authentication.

The setup performed here generated a bcrypt `$2a$` entry with Python bcrypt because `htpasswd` was unavailable. An initial `$2b$` entry failed with `Unrecognized hash type`; `$2a$` succeeded. The secret key is `htpasswd`; OAuth references `htpasswd.fileData.name`, not `htpasswd.file.name`. Passwords and hashes must not be committed.

```bash
# Use a separate kubeconfig so admin credentials/context remain unchanged.
oc --kubeconfig=operator.kubeconfig login \
  https://api.luke.syangsao.net:6443 -u operator
oc --kubeconfig=operator.kubeconfig whoami
# operator
```

For password changes, retrieve the current secret into a protected file, update with `htpasswd -B users.htpasswd operator` (without `-c`), then replace the secret:

```bash
oc get secret htpasswd -n openshift-config -o jsonpath='{.data.htpasswd}' \
  | base64 --decode > users.htpasswd
htpasswd -B users.htpasswd operator
oc create secret generic htpasswd -n openshift-config \
  --from-file=htpasswd=users.htpasswd --dry-run=client -o yaml \
  | oc replace -f -
rm users.htpasswd
```

## 2. Create the OpenShift group

HTPasswd authenticates users; it does not define groups. As administrator:

```bash
oc adm groups new ops operator
# For a group that already exists:
# oc adm groups add-users ops operator
oc get group ops -o jsonpath='{.users}{"\n"}'
# ["operator"]
```

## 3. Grant namespaced migration access

```bash
oc apply -f manifests/ops-migration-rbac.yaml
oc apply -f manifests/ops-migration-target-rbac.yaml
```

| Namespace | Role / RoleBinding | Resources | Verbs | Purpose |
|---|---|---|---|---|
| openshift-mtv | ops-migration-starter | plans.forklift.konveyor.io, resourceNames: windows-test-perms | get | Read the prepared plan |
| openshift-mtv | ops-migration-starter | migrations.forklift.konveyor.io | create | Submit migration |
| openshift-mtv | ops-migration-starter | migrations.forklift.konveyor.io | get, list, watch | Monitor migrations |
| virt-windows | ops-migration-target | virtualmachines.kubevirt.io | create | Pass MTV migration admission authorization |

Both RoleBindings target Group `ops`. No cluster-admin, namespace edit/admin, provider-secret access, target pod/PVC creation or SCC grant was added. The controller performs migration work with its own credentials.

**Important scope limitation:** Kubernetes RBAC cannot restrict `create migrations` by `spec.plan.name` or by `resourceNames`. This grant can submit migrations for other plans in `openshift-mtv` where admission checks permit them. `get` on the plan is name-restricted, but that does not restrict migration submission. Strict single-plan enforcement requires an additional admission policy or namespace isolation. Target `create virtualmachines` also permits direct VM creation in `virt-windows`. Monitoring grants expose other Migration objects in `openshift-mtv`. These are not single-plan-only permissions.

## 4. Verify admission without starting a migration

Before the grants, `operator` could not create migrations. With only the starter RoleBinding, server-side dry run was rejected by the MTV webhook:

```text
Action is forbidden: User "operator" cannot create resource "virtualmachines" in API group "kubevirt.io" in the namespace "virt-windows"
```

After adding only target VM `create`, the same dry run passed. The test also passed using the real authenticated operator context, not just impersonation:

```bash
cat <<'EOF' | oc --kubeconfig=operator.kubeconfig create --dry-run=server -f -
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata:
  generateName: windows-test-perms-
  namespace: openshift-mtv
spec:
  plan:
    name: windows-test-perms
    namespace: openshift-mtv
EOF
# migration.forklift.konveyor.io/windows-test-perms-vxhtb created (server dry run)
```

Administrator negative check:

```bash
oc --as=operator --as-group=ops auth can-i get secrets -n openshift-mtv
# no
```

Dry run proves API authorization and current MTV admission acceptance, not successful disk transfer or console usability. No actual migration was started by these tests. No provider/map/storage/NAD read grants were needed for this API submission path. Console browsing/editing is a different workflow and was not tested.

## 5. Start and monitor when ready

Remove `--dry-run=server` from the preceding `oc create` command to actually start migration. Cold migration can power off the source VM; review the plan before doing so.

```bash
oc --kubeconfig=operator.kubeconfig get plans.forklift.konveyor.io \
  windows-test-perms -n openshift-mtv -o yaml
oc --kubeconfig=operator.kubeconfig get migrations.forklift.konveyor.io \
  -n openshift-mtv -w
```

The plan was Ready, with warnings for automatic target-name normalization and missing VMware guest IP information. Successful admission does not resolve those warnings or prove migration completion.

## 6. Actual migration test as operator

On 2026-11-12, authenticated `operator` (member of `ops`) created a real Migration, without impersonation or additional grants:

```bash
oc --context=openshift-virtualization-os-images/api-luke-syangsao-net:6443/operator whoami
# operator
cat <<'EOF' | oc --context=openshift-virtualization-os-images/api-luke-syangsao-net:6443/operator create -f -
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata:
  generateName: windows-test-perms-ops-
  namespace: openshift-mtv
spec:
  plan:
    name: windows-test-perms
    namespace: openshift-mtv
EOF
# migration.forklift.konveyor.io/windows-test-perms-ops-mfg6k created
```

The same operator context could read Migration progress. Administrator credentials were used only to inspect target pods, VM state, and controller logs; they were not used to submit the migration.

| Milestone | Observed UTC time |
|---|---|
| Migration started | 21:50:26 |
| Disk allocation completed | 21:50:52 |
| Image conversion completed | 21:52:42 |
| Disk transfer completed (102400/102400 MB) | 22:05:41 |
| Target VM creation completed | 22:05:44 |

Target VM `virt-windows/windows2025` was observed Running. The conversion container exited 0 with no restarts. No `forbidden` or `permission denied` messages appeared in the checked controller log window. No permissions were added during the real test.

The user subsequently reported completion. The API snapshot captured at that point still showed `Running` / `WaitForGuestReboots`, rather than a terminal `Succeeded` condition. These are distinct observations: operator submission, disk transfer, and target VM creation are verified; terminal success was not yet established by that snapshot. Guest application functionality and console-user workflows were not tested.

Conclusion: the documented group RoleBindings permit this existing-plan API migration workflow, including real submission and controller execution through target VM creation. A Windows reboot wait is not evidence of missing operator permissions.

## 7. Repository quick reference

Run the setup commands above from this repository root. `manifests/oauth.yaml` configures authentication; it does not contain credentials. The two RBAC manifests grant access to Group `ops`.

After setup, use `oc --kubeconfig=operator.kubeconfig create --dry-run=server -f manifests/migration.yaml` for admission testing. To actually migrate, omit `--dry-run=server`. Do not apply all manifests together: a Migration submission is an intentional, separate operation.

Keep `operator.kubeconfig` and local password files outside version control.
