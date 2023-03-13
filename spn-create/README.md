## Introduction:  
An SPN is a unique identifier that's used to represent a service instance. In Azure, it's used to authenticate against different services without the need for an interactive user login.

**use case:**
If you are getting started with Azure DevOps Pipeline for Application Deployment in Azure, you need two basic things for your Azure Pipeline to Authenticate and deploy to Azure. One — a Service Principal Name (SPN) in Azure and a Service Connection in Azure DevOps.
you can also refer this blog to integrate spn with ADO and postman <link for the blogs>


## Step 1: Sign in to Azure Portal
First, sign in to the [Azure portal](https://portal.azure.com/)  using your credentials.

## Step 2: Create a new Azure AD Application
Navigate to the Azure Active Directory (AD) pane in the portal and create a new Azure AD Application. You can do this by selecting the "App registrations" option in the left-hand pane and then clicking on "New registration" at the top.

### Step 2.1: Select **Azure Active Directory** from the left-hand side menu.
![1.create-service-principal-azure-aad.png](https://github.com/PiyushMittl/Others/blob/main/spn-create/images/1.create-service-principal-azure-aad.png)

### Step 2.2: Select **App registrations** and + **New registration**
![2.create-service-principal-azure-new-reg.png](https://github.com/PiyushMittl/Others/blob/main/spn-create/images/2.create-service-principal-azure-new-reg.png)

### Step 2.3: Enter a name for the **application** (the service principal name).

### Step 2.4: Select **Accounts in this organizational directory only**.

### Step 2.5: (Optional, leave it blank) For **Redirect URI** select **Web** and enter any URL you want; it doesn't have to be real or work.

### Step 2.6: Then select **Register**.
<3.create-service-principal-azure-register.png>
<4.create-service-principal-azure-new-app.png>

## Step 3: Create a Client Secret
Next, you'll need to create a client secret for the application. This is used to authenticate requests made by the application.

### 3.1 Select Add a certificate or secret
<5.create-service-principal-azure-add-secret.png>

### 3.2 Provide a Description and set the Expires for the secret
<6.create-service-pricipal-azure-add-secret-decexpir.png>
<7.create-service-pricipal-azure-add-secret-decexpir-created.png>

### 3.3 Copy the value of Client credentials from Overview
<8.create-service-principal-azure-client-cred.png>

note: 
Make a note of the value of the client secret as you won't be able to access it again once you leave the page.
you can also save these credentials to Key vault Navigate to your **Key vault** -> Select **Settings** --> **Secrets** --> **+ Generate/Import** -> Enter the **Name** of your choice and **Value** as the **Client secret** from your Service Principal -> **Create**

## Step 4:Assign a role to the SPN
why role assignment is required 
Assigning a role to a Service Principal Name (SPN) is an important step in securing access to Azure resources. By assigning a role, you can control the level of access that the SPN has to resources within your Azure subscription. This ensures that only the necessary actions are performed by the SPN and that there is no unauthorized access to sensitive resources.

### Step 4.1 Sign-in to the Azure portal
### Step 4.2 Select the Subscription
Select the level of scope you wish to assign the application to. For example, to assign a role at the subscription scope, search for and select Subscriptions. If you don't see the subscription you're looking for, select global subscriptions filter. Make sure the subscription you want is selected for the tenant. Select your Subscription.
<9.assign-role-spn-select-subs.png>

### Step 4.3 Navigate to the Access Control (IAM) pane
In the left-hand pane, select the "Access Control (IAM)" option.
<10.assign-role-spn-select-subs-IAM.png>

### Step 4.4 Click on "Add"
Click on the "Add" button at the top of the page to add a new role assignment.
<11.assign-role-spn-add-role-asgmnt.png>

### Step 4.5 Choose Privileged administrator roles
locate **Assignment type** (it should be 1st tab) and select Privileged administrator roles and click next.
<12.assign-role-spn-choose-assignment-type.png>

### Step 4.6 Choose a role
In the Role tab, select the role you wish to assign to the application in the list. Choose the role that you want to assign to your SPN. For example, you can select "Contributor" if you want the SPN to have contributor access to a particular resource.
<13.assign-role-spn-choose-role.png>

### Step 4.7 Choose a scope
Choose the scope at which you want to assign the role.On the Members tab. Select Assign access to, then select User, group, or service principal.
<14.assign-role-spn-select-assignaccessto.png>

### Step 4.8 Select Select members. 
By default, Azure AD applications aren't displayed in the available options. To find your application, Search for it by its name.

### Step 4.9 Search SPN and "Review + create"
Click on the "Review + create" button at the bottom of the page once you've filled in all the necessary fields.
<14.assign-role-spn-select-assignaccessto.png>

### Step 4.10 Review and create the role assignment
Review the details of the role assignment and then click on the "Create" button at the bottom of the page to create the role assignment.
<15.assign-role-spn-select-reviewassign.png>

That's it! You've successfully assigned a role to the SPN you created in Azure. Now the SPN will have the permissions associated with the assigned role at the selected scope.
