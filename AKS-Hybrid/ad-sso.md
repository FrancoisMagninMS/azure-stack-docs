---
title: Use Active Directory single sign-on for a secure connection to Kubernetes API server in AKS on Azure Stack HCI and Windows Server
description: Use Active Directory Authentication to securely connect to the API server with SSO credentials
author: sethmanheim
ms.topic: how-to
ms.date: 09/30/2022
ms.author: sethm 
ms.lastreviewed: 1/14/2022
ms.reviewer: sulahiri

# Intent: As an IT Pro, I want to ue Active Directory Authentication to securely connect to the Kubernetes API server with SSO credentials.
# Keyword: secure connection to Kubernetes API server

---

# Use Active Directory single sign-on for a secure connection to Kubernetes API server in Azure Kubernetes Service on Azure Stack HCI and Windows Server

> Applies to: Azure Stack HCI and Windows Server

You can create a secure connection to Kubernetes API server using Active Directory (AD) single sign-on (SSO) credentials. 

Without Active Directory authentication, users must rely on a certificate-based _kubeconfig_ file when connecting to the API server via the `kubectl` command. The _kubeconfig_ file contains secrets such as private keys and certificates that need to be carefully distributed, which can be a significant security risk.

As an alternative to using certificate-based _kubeconfig_, you can use AD SSO sign-on (SSO) credentials as a secure way to connect to the API server. AD integration with Azure Kubernetes Service (AKS) on Azure Stack HCI and Windows Server lets users on a Windows domain-joined machine connect to the API server (via `kubectl`) using their SSO credentials. This removes the need to manage and distribute certificate-based _kubeconfig_ files that contain private keys.

AD integration uses AD _kubeconfig_, which is distinct from the certificate-based _kubeconfig_ files and doesn't contain any secrets. However, the certificate-based _kubeconfig_ file can be used for backup purposes, such as troubleshooting, if there are issues with connecting using Active Directory credentials.

Another security benefit with AD integration is that the users and groups are stored as [security identifiers (SIDs)](/troubleshoot/windows-server/identity/security-identifiers-in-windows). Unlike group names, SIDs are immutable and unique and therefore present no naming conflicts.

> [!Note] 
> Currently, AD SSO connectivity is only supported for workload clusters. 

This document will guide you through the following steps to set up Active Directory as the identity provider and to enable SSO via `kubectl`:

- Create the AD account for the API server, and then create the [keytab](https://web.mit.edu/kerberos/krb5-devel/doc/basic/keytab_def.html) file associated with the account. See [Create AD Auth using the keytab file](#create-ad-auth-using-the-keytab-file) to create the AD account and generate the keytab file.
- Use the [keytab](https://web.mit.edu/kerberos/krb5-devel/doc/basic/keytab_def.html) file to install AD Auth on the Kubernetes cluster. As part of this step, a default role-based access control (RBAC) configuration is automatically created.
- Convert group names to SIDs and vice-versa when creating or editing RBAC configurations, see [create and update the AD group role binding](#create-and-update-the-ad-group-role-binding) for instructions.
 
## Before you begin

Before you start the process of configuring Active Directory SSO credentials, you should ensure you have the following items available:

 - The latest **Aks-Hci PowerShell** module is installed. If you need to install it, see [download and install the AksHci PowerShell module](./kubernetes-walkthrough-powershell.md#install-the-akshci-powershell-module). 
 - The AKS host is installed and configured. If you need to install the host, follow the steps to [configure your deployment](./kubernetes-walkthrough-powershell.md#step-3-configure-your-deployment).  
 - The keytab file is available to use. If it isn't available, follow the steps to [create the API server AD account and the keytab file](#create-the-api-server-ad-account-and-the-keytab-file). 
 
   > [!NOTE]
   > You should generate the keytab file for a specific service principal name (SPN), and this SPN must correspond to the API server AD account for the workload cluster. You must also ensure that the same SPN is used throughout the AD authentication process. The keytab file should be named _current.keytab_.

 - One API server AD account is available for each AKS on Azure Stack HCI and Windows Server workload cluster.
 - The client machine must be a Windows domain-joined machine.

## Create AD Auth using the keytab file

### Step 1: Create the workload cluster and enable AD authentication

Before you install AD authentication, you must first create an AKS on Azure Stack HCI and Windows Server workload cluster and enable the AD authentication add-on on the cluster. If you don't enable AD authentication when you create the new cluster, you won't be able to enable it later.

Open PowerShell as an administrator and run the following using the **-enableADAuth** parameter of the `New-AksHciCluster` command:

```powershell
New-AksHciCluster -name mynewcluster1 -enableADAuth
```

For each workload cluster, ensure there's one API server AD account available.

For details on creating the workload cluster, see [create Kubernetes clusters using Windows PowerShell](./kubernetes-walkthrough-powershell.md).

### Step 2: Install AD authentication

Before you can install AD authentication, the workload cluster must be installed and the AD authentication enabled on the cluster. To install AD authentication, use one of the following options.

#### Option 1

For a domain-joined Azure Stack HCI and Windows Server cluster, open PowerShell as an administrator and run the following command:

```powershell
Install-AksHciAdAuth -name mynewcluster1 -keytab .\current.keytab -SPN k8s/apiserver@CONTOSO.COM -adminUser contoso\bob
```  

> [!NOTE]
> For `SPN k8s/apiserver@CONTOSO.com`, use the format SPN k8s/apiserver@\<realm name>\. On first attempt, specify \<realm name> as uppercase. However, if you are having issues, create the SPN with lowercase letters. Kerberos is case sensitive.

#### Option 2

If the cluster host isn't domain-joined, use the admin user name or group name in SID format as shown in the example below:
 
```powershell
Install-AksHciAdAuth -name mynewcluster1 -keytab .\current.keytab -SPN k8
```  

To find the SID for the user account, see [Determine the user or group security identifier](#determine-the-user-or-group-security-identifier). 

Before proceeding to the next steps, you should make note of the following items:

- Make sure the keytab file is named _current.keytab_.
- Replace the SPN that corresponds to your environment.
- The **-adminUser** parameter creates a corresponding role binding for the specified AD group with cluster admin privileges. Replace _contoso\bob_ (as shown in Option 1 above) with the AD group or user that corresponds to your environment.

### Step 3: Test the AD webhook and keytab file

You need to make sure the AD webhook is running on the API server and the keytab file is stored as a Kubernetes secret. To get the certificate-based _kubeconfig_ file for the workload cluster, follow these steps:

1. Get a certificate-based _kubeconfig_ file using the following command. You'll use the _kubeconfig_ file to connect to the cluster as a local host.

   ```powershell
   Get-AksHciCredential -name mynewcluster1
   ```  

2. Run `kubectl` on the server that you connected to (using the certificate-based _kubeconfig_ file you previously created) and then check the AD webhook deployment to make sure it's in the format _ad-auth-webhook-xxxx_.  

    ```bash
    kubectl get pods -n=kube-system
    ```

3. Run `kubectl` to check that the keytab file is deployed as a secret and is listed as a Kubernetes secret: 

   ```bash
   kubectl get secrets -n=kube-system
   ```

### Step 4: Create the AD kubeconfig file

Once the AD webhook and keytab are successfully deployed, create the AD _kubeconfig_ file.
After the file is created, copy the AD _kubeconfig_ file to the client machine and use it to authenticate to the API server. Unlike the certificate-based _kubeconfig_ file, AD _kubeconfig_ isn't a secret and is safe to copy as plain text.

Open PowerShell as an administrator and run the following command:  

   ```powershell
   Get-AksHciCredential -name mynewcluster1 -configPath .\AdKubeconfig -adAuth
   ```

### Step 5: Copy kubeconfig and other files to the client machine

You should copy the three files listed below from the Azure Stack HCI cluster to your client machine:

- Copy the AD _kubeconfig_ file created in the previous step to $env:USERPROFILE\.kube\config.

- Create the folder path `c:\adsso` and copy the following files from the Azure Stack HCI cluster to your client machine.
  - Kubectl.exe under `$env:ProgramFiles\AksHci` to c:\adsso
  - Kubectl-adsso.exe under `$env:ProgramFiles\AksHci` to c:\adsso

  > [!NOTE]
  > The adsso.exe file is generated on the server when you run the `Get-AksHciCredential` cmdlet.

### Step 6: Connect to the API server from the client machine

After you've completed the previous steps, use your SSO credentials to sign in to your Windows domain-joined client machine. Open PowerShell, and then attempt to access the API server using `kubectl`. If you're successfully able to complete the operation, you have set up AD SSO correctly.

## Create and update the AD group role binding

As mentioned in Step 2, a default role binding with cluster admin privileges is created for the user and/or the group that was provided during installation. Role binding in Kubernetes defines the access policies for AD groups. This step describes how to use RBAC to create new AD group role bindings in Kubernetes and to edit existing role bindings. For example, the cluster admin may want to grant additional privileges to users by using AD groups (which makes the process more efficient). To learn more about RBAC, see [using RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

When creating or editing more AD group RBAC entries, the subject name should be pre-fixed by "`microsoft:activedirectory:CONTOSO\\group name`". Note that the names must contain a domain name and a prefix that are enclosed by double quotes.

Here are two examples:

**Example 1**

```bash
apiVersion: rbac.authorization.k8s.io/v1 
 kind: ClusterRoleBinding 
 metadata: 
   name: ad-user-cluster-admin 
 roleRef: 
   apiGroup: rbac.authorization.k8s.io 
   kind: ClusterRole 
   name: cluster-admin 
 subjects: 
 - apiGroup: rbac.authorization.k8s.io 
   kind: User 
   name: "microsoft:activedirectory:CONTOSO\Bob" 
```

**Example 2**

The example below shows how to create a custom role and role binding for a [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) with an AD group. In the example, "SREGroup" is a pre-existing group in the Contoso Active Directory. When users are added to the AD group, they're immediately granted privileges.

```bash
kind: Role 
 apiVersion: rbac.authorization.k8s.io/v1 
 metadata: 
   name: sre-user-full-access 
   namespace: sre 
 rules: 
 - apiGroups: ["", "extensions", "apps"] 
   resources: ["*"] 
   verbs: ["*"] 
 - apiGroups: ["batch"] 
   resources: 
   - jobs 
   - cronjobs 
   verbs: ["*"] 
 apiVersion: rbac.authorization.k8s.io/v1 
 kind: RoleBinding 
 namespace: sre 
 metadata: 
   name: ad-user-cluster-admin 
 roleRef: 
   apiGroup: rbac.authorization.k8s.io 
   kind: Role 
   name: sre-user-full-access 
 subjects: 
 - apiGroup: rbac.authorization.k8s.io 
   kind: User 
   name: "microsoft:activedirectory:CONTOSO\SREGroup" 
```

Before applying the yaml file, the **group and user names** should always be converted to SIDs using the command:

```bash
kubectl-adsso nametosid <rbac.yml>
```  

Similarly, in order to update an existing RBAC, you can convert the SID to a user-friendly group name before making changes. To convert the SID, use the command: 

```bash
kubectl-adsso sidtoname <rbac.yml>
```

## Change the AD account password associated with the API server account

When the password is changed for the API server account, you must uninstall the AD authentication add-on and reinstall it using the updated current and previous keytabs. 

Every time you change the password, you must rename the current keytab (_current.keytab_) to _previous.keytab_.  Then, make sure you name the new password _current.keytab_.

> [!Important] 
> The files must be named _current.keytab_ and _previous.keytab_, respectively. The existing role bindings are not affected by this.

### Uninstall and reinstall AD authentication

You may want to reinstall AD SSO when the account for the API server is changed, when the password is updated, or to troubleshoot a failure.

To uninstall AD authentication, open PowerShell as an administrator and run the following command:  

```powershell
Uninstall-AksHciAdAuth -name mynewcluster1
```

To reinstall AD authentication, open PowerShell as an administrator and run the following command:

```powershell
Install-AksHciAdAuth -name mynewcluster1 -keytab <.\current.keytab> -previousKeytab <.\previous.keytab> -SPN <service/principal@CONTOSO.COM> -adminUser CONTOSO\Bob
```

> [!Note] 
> The `-previousKeytab` parameter is required only when you change the password. This is to avoid downtime if the clients have cached tickets.

## Create the API server AD Account and the keytab file

There are two steps involved in this process. First, create a new AD account/user for the API server with the service principal name (SPN) and second, create a keytab file for the AD account.

### Step 1: Create a new AD account or user for the API server

Use the [New-ADUser](/powershell/module/activedirectory/new-aduser) PowerShell command to create a new AD account/user with the SPN. Here's an example: 

```powershell 
New-ADUser -Name apiserver -ServicePrincipalNames k8s/apiserver -AccountPassword (ConvertTo-SecureString "password" -AsPlainText -Force) -KerberosEncryptionType AES128 -Enabled 1
```

### Step 2: Create the keytab file for the AD account

To create the keytab file, you use the [ktpass](/windows-server/administration/windows-commands/ktpass) Windows command. 

Here's an example using ktpass:

```bash
ktpass /out current.keytab /princ k8s/apiserver@CONTOSO.COM /mapuser contoso\apiserver_acct /crypto all /pass p@$$w0rd /ptype KRB5_NT_PRINCIPAL
```

> [!NOTE]
> If you see this error, **DsCrackNames returned 0x2 in the name entry**, ensure the parameter for `/mapuser` is in form `mapuser DOMAIN\user` where DOMAIN is the output of echo `%userdomain%`.

## Determine the user or group security identifier

Use one of the following two options to find the SID for your account or other accounts.

To find the SID associated with your account, from a command prompt of your home directory, type the following:

```bash
whoami /user
``` 

To find the SID associated with another account, open PowerShell as an administrator and run the following:

```powershell
(New-Object System.Security.Principal.NTAccount(<CONTOSO\Bob>)).Translate([System.Security.Principal.SecurityIdentifier]).value
```

## Troubleshooting (certificates)

The webhook and the API server use certificates to mutually validate the TLS connection. This certificate expires in 500 days. To verify that the certificate has expired, view the logs from an `ad-auth-webhook` container:

```bash
kubectl logs ad-auth-webhook-xxx
``` 

If you see certificate validation errors, complete the steps to [uninstall and reinstall the webhook](ad-sso.md#uninstall-and-reinstall-ad-authentication) and get new certificates.

## Clean up and best practices

- Use a unique account for each cluster.
- Don't reuse the password for the API server account across clusters.
- Delete the local copy of the keytab file as soon as you create the cluster and verify that the SSO credentials work.
- Delete the Active Directory user that was created for the API server. For more information, see [Remove-ADUser](/powershell/module/activedirectory/remove-aduser?view=windowsserver2019-ps&preserve-view=true).

## Next steps 

In this how-to guide, you learned how to configure AD Authentication to securely connect to the API server with SSO credentials. Next, you can:

- [Deploy Linux applications on a Kubernetes cluster](./deploy-linux-application.md).
- [Deploy a Windows Server application on a Kubernetes cluster](./deploy-windows-application.md).
