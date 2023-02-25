## Introduction:  
Integrating Postman with Azure DevOps (ADO) can streamline your API testing and automation process, helping you to save time and improve your software quality. Postman is a popular tool for testing APIs, allowing you to create and run tests in an easy-to-use interface. Azure DevOps is a cloud-based service that offers a suite of tools for agile software development, including version control, continuous integration and deployment, and project management. 

By integrating Postman with Azure DevOps, you can run tests as part of your development pipeline, automate testing, and collaborate more effectively with your team. In this blog post, we'll walk you through the steps of integrating Postman with Azure DevOps and show you how to use it to improve your development workflow.

## Installation and Setup: 
1. Download and install Postman from the official website: https://www.postman.com/downloads/
2. Once the installation is complete, open Postman and create a new collection to hold your API tests.

## Write test cases and export collection:
1. Open Postman and create a new request by clicking the New button in the upper left-hand corner of the screen.
2. In the new request tab, enter the request URL and HTTP method. You can also add headers, parameters, and a request body as needed.
3. Once you have created the request, click the Tests button next to the Send button.
4. In the Tests tab, you can write JavaScript code to test the response from the API. Here's an example test case to check if the response status is 200 OK:
```
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});
```
5. You can add more test cases to the same request by writing additional code in the Tests tab. For example, you can check if the response body contains a certain value:
```
pm.test("Response body contains 'success'", function () {
    pm.expect(pm.response.text()).to.include("success");
});
```
6. Once you have written your test cases, click the Send button to run the request and see the results of your tests.
7. If all of your test cases pass, you'll see a green checkmark next to each one in the Tests tab. If any of your test cases fail, you'll see a red X next to the failing test.  
  
To save postman collection follow the below steps   
  
8. Open Postman and select the collection you want to export.
9. Click on the "..." icon next to the collection name and select "Export".
10. In the "Export" window, choose the format in which you want to export the collection. The available formats are Postman Collection v1, Postman Collection v2, and OpenAPI 3.0.
11. Choose the folder where you want to save the exported file and click on "Save".

note:If you’re using any tokens or variables, you can using Postman environments, and you need to export that as well.

## Run postman collection in ADO 

1. First, create a new pipeline or open an existing pipeline in Azure DevOps.
2. Add a new task to the pipeline by clicking on the "+" icon next to the pipeline stages.
3. In the "Add tasks" dialog box, search for "Npm" in the search bar.
4. Select the "Npm" task from the search results and click on "Add".
5. Configure the Npm task by specifying the following information:
```
Display name
task: Npm@1
inputs:
    command: custom
    customCommand:  install -g newman
```
6. Add Cmdline task like above, Configure this Command Line task by specifying the following information:
```
Display name
task: CmdLine@2
inputs: 
    script: newman run <postman_collection_path>/<collection_file_name>.json -x -r junit --reporter-junit-export <directory_to_save_report>/<report_file_name>.xml
```
7. Add PublishTestResults task like above, Configure this Publish Test Result task by specifying the following information:
Display name
task: PublishTestResults@2
inputs:
    testResultsFormat: 'JUnit'
    testResultsFiles: <directory_to_save_report>/<report_file_name>.xml
8. Save the task and run the pipeline.

Example:
```
    - task: Npm@1
      inputs:
        command: 'custom'
        customCommand: 'install newman -g'

    - task: AzureCLI@2
      inputs:
        azureSubscription: '*****'
        scriptType: 'pscore'
        scriptLocation: 'inlineScript'
        inlineScript: |
          newman run ./test_postman_collection.json -e ./test2.postman_environment.json -x -r cli,junit --reporter-junit-export $(Build.SourcesDirectory)/.pipelines/JunitResults.xml --verbose

    - task: PublishTestResults@2
      displayName: Publish Test Results
      inputs:
        testResultsFormat: 'JUnit'
        testResultsFiles: $(Build.SourcesDirectory)/.pipelines/JunitResults.xml
```      


The command we provided is running a tool called Newman which is used to run Postman collections and generate reports. Here's a breakdown of each part of the command:

```
./test_postman_collection.json: This is the path to the Postman collection file that Newman will run.
-e ./test2.postman_environment.json: This specifies the environment file to be used for the Postman collection. An environment file contains variables that can be used in requests within the collection. In this case, the test2.postman_environment.json file is being used.
-x: This flag tells Newman to generate an XML report.
-r cli,junit: This flag specifies which reporters to use. In this case, the cli reporter is being used for console output, and the junit reporter is being used to generate an XML report.
--reporter-junit-export $(Build.SourcesDirectory)/.pipelines/JunitResults.xml: This specifies the file path and name for the XML report that Newman will generate. In this case, the report will be saved to JunitResults.xml in the .pipelines directory of the build sources directory. The $(Build.SourcesDirectory) is a variable that is likely being used in a build pipeline to dynamically specify the directory path.
--verbose: This flag specifies that Newman should output more verbose logging information during the run.
```
## View test results 

You can find more details about the test run in the Test Plans > Runs.

Locate the Summary tab and find the Test Summary executed throught newman
![Test Summary](https://github.com/PiyushMittl/Others/blob/main/postman-integration-ado/images/5.test_sumary.jpg)

Locate the Result tab and find the Test Results executed throught newman
![Test Results](https://github.com/PiyushMittl/Others/blob/main/postman-integration-ado/images/6.test_result.jpg)

