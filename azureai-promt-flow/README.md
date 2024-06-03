## Introduction:  
In today's digital era, chatbots have become an essential tool for businesses, enabling them to offer instant customer service and enhance user engagement. However, building a chatbot that can interact intelligently with users based on a diverse range of text data formats (such as HTML, TXT, PDF, etc.) can be a daunting task. Fortunately, Azure provides a suite of powerful AI services that simplify this process.

In this blog, we will walk you through the steps of creating a versatile chatbot using Azure. This chatbot will be capable of processing any text data you provide, converting it into a format that it can understand, and using that data to engage in meaningful interactions. We'll leverage Azure's Cognitive Services to extract and analyze text from various formats and store this processed information in a vector database. By the end of this guide, you will have a fully functional chatbot that can interact with users based on the text data you've provided, making use of cutting-edge AI technologies provided by Azure.

Let’s dive in and explore how you can harness the power of Azure to build a sophisticated, intelligent chatbot capable of handling a wide array of text inputs.

## Prerequisite: 
1. Azure Subscription 
2. Resource Group
3. GPT3.5 or 4 Access
4. Cognitive Search Service (aka. Azure AI Search)
5. Azure Machine Learning Service
6. Compute

## Setup to create Azure Machine Learning Prompt Flow: 
1. Create Resource Group  
2. Create Azure ML Resource  
3. Create Azure AI Resource  
4. Create Model Deployments  
5. Create a Compute Instance  
6. Azure Open AI  
7. Cognitive Search   
8. Azure ML Pipeline  
9. Create Prompt Flow  
10. Generate Prompt Text  
11. Deploy  
12. Testing  
   
## Step1: Create Resource Group
1.1 On the Azure Portal homepage, locate and click on the Marketplace section in the left-hand navigation pane.  
1.2 In the Marketplace, use the search bar to type "resource group".  
1.3 Press Enter to perform the search.  
1.4 From the search results, locate the Resource group option listed as an Azure Service by Microsoft.  
1.5 Click on the Create button located under the Resource group option.  
1.6 Choose the Azure subscription under which you want to create the resource group.  
1.7 Enter a unique name for your resource group.  
1.8 Select the appropriate Azure region where you want the resource group to be located.  
1.9 Click the Review + create button.  
![1.1.AzureAIPromptFlow_create_RG.jpg](../azureai-promt-flow/images/1.1.AzureAIPromptFlow_create_RG.png)

I have created resource group named ```blog-rg```  
after successful resource creation you would see your resource group created as given below:  
![1.2.AzureAIPromptFlow_create_RG.jpg](../azureai-promt-flow/images/1.2.AzureAIPromptFlow_create_RG.png)

## Step2: Create Azure AI Resource
Follow the steps mentioned from 1.1 till 1.9 (instead of searching "resource group" search "Azure AI Search")
![2.1.AzureAIPromptFlow_create_AISearch.jpg](../azureai-promt-flow/images/2.1.AzureAIPromptFlow_create_AISearch.png)

I have created resource group named ```blog-rg``` 
after successful resource creation you would see your resource group created as given in below reference image:
![2.1.AzureAIPromptFlow_create_AISearch](../azureai-promt-flow/images/2.2.AzureAIPromptFlow_create_AISearch.png)

![3.AzureAIPromptFlow_create_AISearch.png](../azureai-promt-flow/images/3.AzureAIPromptFlow_create_AISearch.png)


## Step3: Create Azure ML Resource

![4.1.AzureAIPromptFlow_create_searchcreate_AzureMachineLearning.jpg](../azureai-promt-flow/images/4.1.AzureAIPromptFlow_create_searchcreate_AzureMachineLearning.png)
![4.2.AzureAIPromptFlow_create_searchcreate_AzureMachineLearning.jpg](../azureai-promt-flow/images/4.2.AzureAIPromptFlow_create_searchcreate_AzureMachineLearning.png)

## Step4: Create Model Deployments 

![5.1.AzureAIPromptFlow_create_searchcreate_AzureOpenAI.png](../azureai-promt-flow/images/5.1.AzureAIPromptFlow_create_searchcreate_AzureOpenAI.png)

![](../azureai-promt-flow/images/5.2.AzureAIPromptFlow_create_searchcreate_AzureOpenAI.png)
![](../azureai-promt-flow/images/5.3.AzureAIPromptFlow_AzureOpenAI_gotoopenaistudio.png)
![](../azureai-promt-flow/images/5.4.AzureAIPromptFlow_AzureOpenAI_create_deployment.png)
![](../azureai-promt-flow/images/5.5.AzureAIPromptFlow_AzureOpenAI_create_deployment.png)
![](../azureai-promt-flow/images/5.6.AzureAIPromptFlow_AzureOpenAI_create_deployment.png)
![](../azureai-promt-flow/images/5.7.AzureAIPromptFlow_AzureOpenAI_create_deployment.png)
![](../azureai-promt-flow/images/5.8.AzureAIPromptFlow_AzureOpenAI_create_deployment.png)

## Step5: Create a Compute Instance 
![](../azureai-promt-flow/images/6.1.AzureAIPromptFlow_AzureOpenAI_create_compute.png)
![](../azureai-promt-flow/images/6.2.AzureAIPromptFlow_AzureOpenAI_create_compute.png)
![](../azureai-promt-flow/images/6.3.AzureAIPromptFlow_AzureOpenAI_create_compute.png)
![](../azureai-promt-flow/images/6.4.AzureAIPromptFlow_AzureOpenAI_create_compute.png)
![](../azureai-promt-flow/images/6.5.AzureAIPromptFlow_AzureOpenAI_create_compute.png)
![](../azureai-promt-flow/images/6.6.AzureAIPromptFlow_AzureOpenAI_create_compute.png)

## Step6: Azure Open AI 
![](../azureai-promt-flow/images/7.1.AzureAIPromptFlow_promptflow_connection_create.png)
![](../azureai-promt-flow/images/7.2.AzureAIPromptFlow_promptflow_connection_create.png)
![](../azureai-promt-flow/images/7.3.AzureAIPromptFlow_promptflow_connection_create.png)

## Step7: Cognitive Search 
![](../azureai-promt-flow/images/8.1.AzureAIPromptFlow_promptflow_connection_azais_create.png)
![](../azureai-promt-flow/images/8.2.AzureAIPromptFlow_promptflow_connection_azais_create.png)
![](../azureai-promt-flow/images/8.3.AzureAIPromptFlow_promptflow_connection_azais_create.png)

## Step8: Azure ML Pipeline
![](../azureai-promt-flow/images/9.1.AzureAIPromptFlow_promptflow_vectordb_create.png)
![](../azureai-promt-flow/images/9.2.AzureAIPromptFlow_promptflow_vectordb_create.png)
![](../azureai-promt-flow/images/9.3.AzureAIPromptFlow_promptflow_vectordb_create.png)
![](../azureai-promt-flow/images/9.4.AzureAIPromptFlow_promptflow_vectordb_create.png)
![](../azureai-promt-flow/images/9.5.AzureAIPromptFlow_promptflow_vectordb_create.png)
![](../azureai-promt-flow/images/9.6.AzureAIPromptFlow_promptflow_vectordb_create.png)

## Step9: Create Prompt Flow
![](../azureai-promt-flow/images/10.1.AzureAIPromptFlow_promptflow_flow_create.png)
![](../azureai-promt-flow/images/10.2.AzureAIPromptFlow_promptflow_flow_create.png)
![](../azureai-promt-flow/images/10.3.AzureAIPromptFlow_promptflow_flow_create.png)
![](../azureai-promt-flow/images/10.4.AzureAIPromptFlow_promptflow_flow_create.png)
![](../azureai-promt-flow/images/10.5.AzureAIPromptFlow_promptflow_flow_create.png)
![](../azureai-promt-flow/images/10.6.AzureAIPromptFlow_promptflow_flow_create.png)
![](../azureai-promt-flow/images/10.7.AzureAIPromptFlow_promptflow_flow_create.png)
![](../azureai-promt-flow/images/10.8.AzureAIPromptFlow_promptflow_flow_create.png)

## Step10: Generate Prompt Text
![](../azureai-promt-flow/images/11.1.AzureAIPromptFlow_promptflow_chatwithdsapdf.png)
![](../azureai-promt-flow/images/11.2.AzureAIPromptFlow_promptflow_chatwithdsapdf.png)

## Step11: Deploy
## Step12: Testing