## Introduction:  
Azure DevOps is a powerful platform that provides a suite of tools for software development, including version control, continuous integration, and continuous deployment. One of the most important features of Azure DevOps is its ability to connect to external resources like Azure, GitHub, and other services. In order to establish a connection between Azure DevOps and external resources, you need to create a service connection.

A service connection is a secure way to authenticate connections between Azure DevOps and external resources, allowing you to use features like pipelines, webhooks, and OAuth tokens. In this blog post, we will walk you through the process of creating a service connection in Azure DevOps using Service Principal Names (SPNs), a secure authentication method that provides granular control over resource access. By the end of this tutorial, you will be able to easily create service connections to external resources in Azure DevOps, enabling you to automate your software development and delivery workflows.

## Step 1: Sign in to DevOps portal
Open the Azure DevOps portal and go to the project where you want to create the service connection.

## Step 2: Click on the "Project settings" icon and select the "Service connections" option.
Once you have logged into the Azure DevOps portal and navigated to your project, clicking on the "Project settings" icon, which is located in the bottom-left corner of the screen.

![1.create-sc-ado-spn-projectsettings.png](https://github.com/PiyushMittl/Others/blob/main/create-sc-ado-spn/images/1.create-sc-ado-spn-projectsettings.png)

## Step 3: Click on the "+New service connection" button 
you can create the "New service connection" option by clicking on "Service connection" under "Project Settings" icon.
![2.create-sc-ado-spn-newsc.png](https://github.com/PiyushMittl/Others/blob/main/create-sc-ado-spn/images/2.create-sc-ado-spn-newsc.png)

## Step 4: Choose a Service Connection or Connection type
once you clicked on "New Service Connection" select "Azure Resource Manager" as Service Connection Type and click "Next".
![3.create-sc-ado-spn-newsc-arm.png](https://github.com/PiyushMittl/Others/blob/main/create-sc-ado-spn/images/3.create-sc-ado-spn-newsc-arm.png)

## Step 5: Select Authentication Method
After selecting "Azure Resource Manager" as "Service Connection Type" choose "Authentication Method" as "Service Principal (Manual)" and click "Next".
![4.create-sc-ado-spn-newsc-spnmanual.png](https://github.com/PiyushMittl/Others/blob/main/create-sc-ado-spn/images/4.create-sc-ado-spn-newsc-spnmanual.png)

## Step 6: Fill below details and Authenticate.
Scope Level: Select "Subscription"  
Subscription Id: Fill your Subscription Id for which you have created your SPN.  
Provide below Authentication details  
Service Principal ID: Provide your service principal ID (Enter the client secret for the SPN. This is the password that you generated when you created the SPN)  
Credential: Choose "Service Principal Key" and provide SPN passcode here  
Tenant Id: Provide Tenant Id  

After filling all these details "Authenticate" these details by clicking on "Verify", you should get "Verification Succeeded" message.

![5.create-sc-ado-spn-newsc-testverifysave.png](https://github.com/PiyushMittl/Others/blob/main/create-sc-ado-spn/images/5.create-sc-ado-spn-newsc-testverifysave.png)

You can refer below image to map SPN properties with these Authentication properties  
![5.1.create-sc-ado-spn-newsc-testverifysave.png](https://github.com/PiyushMittl/Others/blob/main/create-sc-ado-spn/images/5.1.create-sc-ado-spn-newsc-testverifysave.png)

## Step 7: Verify and Save the Azure Service Connection.
Once Verification is done click on "Verify and Save" button, now you would see your newly create "Service Connection"
![6.create-sc-ado-spn-newsc-testverifysave-success.png](https://github.com/PiyushMittl/Others/blob/main/create-sc-ado-spn/images/6.create-sc-ado-spn-newsc-testverifysave-success.png)

## Using Service Connection in ADO
Now you can use this created "Service Connection" on ADO tasks directly and you no need to login to AZ CLI because you can manage the Azure resources for the "Subscription" the "Service Connection" was created using "subscription" of SPN. Subscription .
```
    steps:  
    - task: AzureCLI@2
      inputs:
        azureSubscription: 'test-blog-spn'
        scriptType: 'pscore'
        scriptLocation: 'inlineScript'
        inlineScript: |
           <write your AZ cli command without login>
```           

Overall, the "Service connections" option in the "Project settings" menu of Azure DevOps is a powerful tool that enables you to connect your project to external resources and automate your software development and delivery workflows. By following the steps outlined in this tutorial, you can easily create and manage service connections using SPNs, a secure and granular authentication method that provides fine-grained control over resource access.
