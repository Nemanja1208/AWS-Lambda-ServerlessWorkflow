### Documentation source - `https://aws.amazon.com/tutorials/create-a-serverless-workflow-step-functions-lambda/`

# Implementation steps

1.  Create an `` AWS Identity and `Access Management (IAM) `` role

    AWS IAM is a web service that helps you securely control access to AWS resources. In this step, you will create an IAM role that allows Step Functions to access Lambda.

    - Open the `IAM Management Console`. Select Roles from the left navigation pane and then choose Create role.

    - On the Create role screen, under Trusted entity type, keep AWS service selected. Under Use case, search `Step Functions` in the dropdown and select Step Functions. Choose Next.

    - In the Add permissions section, choose Next.

    - In the Name, review, and create section, in Role name, enter step_functions_basic_execution. Choose Create role.

2.  Create a state machine and serverless workflow

    Your first step is to design a workflow that describes how you want support tickets to be handled in your call center. Workflows describe a process as a series of discrete tasks that can be repeated again and again.

    You are able to sit down with the call center manager to talk through best practices for handling support cases. Using the visual workflows in `Step Functions` as an intuitive reference, you define the workflow together.

    Then, you will design your workflow in `AWS Step Functions`. Your workflow will call one AWS Lambda function to create a support case, invoke another function to assign the case to a support representative for resolution, and so on. It will also pass data between Lambda functions to track the status of the support case as it's being worked on.

    -Open the `AWS Step Functions console`. Select Write your workflow in code.

    - Replace the contents of the state machine definition window with the Amazon States Language (ASL) state machine definition below. Amazon States Language is a JSON-based, structured language used to define your state machine.

      This state machine uses a series of Task states to open, assign, and work on a support case. Then, a Choice state is used to determine if the case can be closed or not. Two more Task states then close or escalate the support case as appropriate.

      ```{
      "Comment": "A simple AWS Step Functions state machine that automates a call center support session.",
      "StartAt": "Open Case",
      "States": {
      "Open Case": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:FUNCTION_NAME",
      "Next": "Assign Case"
      },
      "Assign Case": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:FUNCTION_NAME",
      "Next": "Work on Case"
      },
      "Work on Case":{
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:FUNCTION_NAME",
      "Next": "Is Case Resolved"
      },
      "Is Case Resolved": {
      "Type" : "Choice",
      "Choices": [
      {
      "Variable": "$.Status",
      "NumericEquals": 1,
      "Next": "Close Case"
      },
      {
      "Variable": "$.Status",
      "NumericEquals": 0,
      "Next": "Escalate Case"
      }
      ]
      },
      "Close Case": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:FUNCTION_NAME",
      "End": true
      },
      "Escalate Case": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:FUNCTION_NAME",
      "Next": "Fail"
      },
      "Fail": {
      "Type": "Fail",
      "Cause": "Engage Tier 2 Support." }
      }
      }
      ```

    - Choose the refresh button to show the ASL state machine definition as a visual workflow. In our scenario, you can easily verify that the process is described correctly by reviewing the visual workflow with the call center manager. Choose Next.

    - In the State machine name text box, enter CallCenterStateMachine

    - Under Permissions, select Choose an existing role, select step_functions_basic_execution from the dropdown.

    - At the bottom of the page, choose Create state machine.

3.  Create your `Lambda` functions

    Now that you have created your state machine, you can decide how you want it to perform work. You can connect your state machine to Lambda functions and other microservices that already exist in your environment, or create new ones. In this tutorial, you will create a few simple Lambda functions that simulate various steps for handling support calls, such as assigning the case to a customer support representative.

    - Open the `AWS Lambda console`. Choose Create function.

    - Select Author from scratch.

    - Configure your first Lambda function with these settings :

      - Name – OpenCaseFunction.
      - Runtime – Node.js 16.x.
      - Execution role – Create a new role with basic Lambda permissions

    - Choose Create function.

    - Replace the contents of the Function code window with the following code, and then choose Deploy :

      ```
          exports.handler = (event, context, callback) => {
          // Create a support case using the input as the case ID, then return a confirmation message
          var myCaseID = event.inputCaseID;
          var myMessage = "Case " + myCaseID + ": opened...";
          var result = {Case: myCaseID, Message: myMessage};
          callback(null, result);
          };

      ```

    - At the top of the page, select Functions.

    - Repeat steps 3b-3d to create four more `Lambda` functions, using the lambda_basic_execution IAM role you created in step 3c.

    - Define `AssignCaseFunction` as:

      ```
      exports.handler = (event, context, callback) => {
      // Assign the support case and update the status message
      var myCaseID = event.Case;
      var myMessage = event.Message + "assigned...";
      var result = {Case: myCaseID, Message: myMessage};
      callback(null, result);
      ;
      ```

    - Define `WorkOnCaseFunction` as:

      ```
      exports.handler = (event, context, callback) => {
        // Generate a random number to determine whether the support case has been resolved, then return that value along with the updated message.
        var min = 0;
        var max = 1;
        var myCaseStatus = Math.floor(Math.random() * (max - min + 1)) + min;
        var myCaseID = event.Case;
        var myMessage = event.Message;
        if (myCaseStatus == 1) {
            // Support case has been resolved
            myMessage = myMessage + "resolved...";
        } else if (myCaseStatus == 0) {
            // Support case is still open
            myMessage = myMessage + "unresolved...";
        }
        var result = {Case: myCaseID, Status : myCaseStatus, Message: myMessage};
        callback(null, result);
        };
      ```

    - Define `CloseCaseFunction` as:

      ```
      exports.handler = (event, context, callback) => {
      // Close the support case
      var myCaseStatus = event.Status;
      var myCaseID = event.Case;
      var myMessage = event.Message + "closed.";
      var result = {Case: myCaseID, Status : myCaseStatus, Message: myMessage};
      callback(null, result);
      };
      ```

    - Define `EscalateCaseFunction` as:

      ```
      exports.handler = (event, context, callback) => {
      // Escalate the support case
      var myCaseID = event.Case;
      var myCaseStatus = event.Status;
      var myMessage = event.Message + "escalating.";
      var result = {Case: myCaseID, Status : myCaseStatus, Message: myMessage};
      callback(null, result);
      };
      ```

    - When complete, you should have five `Lambda` functions.

4.  Populate your workflow

    The next step is to populate the task states in your Step Functions workflow with the Lambda functions you just created.

    - Open the `AWS Step Functions console`. In the State machines section, select CallCenterStateMachine and choose Edit.

    - Choose Edit on the CallCenterStateMachine screen.

    - In the state machine Definition section, find the line below the Open Case state which starts with Resource.

    - Replace the ARN with the ARN of your OpenCaseFunction.

    - If you choose the sample ARN, a list of the `AWS Lambda` functions in your account will appear and you can select it from the list.

    - Repeat the previous step to update the `Lambda function` ARNs for the Assign Case, Work on Case, Close Case, and Escalate Case task states in your state machine, then choose Save.

5.  Execute your workflow

    Your serverless workflow is now ready to be executed. A state machine execution is an instance of your workflow, and occurs each time a `Step Functions` state machine runs and performs its tasks. Each `Step Functions` state machine can have multiple simultaneous executions, which you can initiate from the `Step Functions console` (which is what you will do next), or by using AWS SDKs, Step Functions API actions, or the AWS CLI. An execution receives JSON input and produces JSON output.

    - Choose Start execution.

    - A New execution dialog box appears. To supply an ID for your support case, enter the following content in the New execution dialog box in the Input window, then choose Start execution. `{"inputCaseID": "001"}`

    - As your workflow executes, each step will change color in the Visual workflow pane. Wait a few seconds for execution to complete. Then, in the Execution details pane, choose Execution input and Execution output to view the inputs and results of your workflow.

    - Scroll down to the Execution event history section. Click through each step of the execution to see how Step Functions called your `Lambda functions` and passed data between functions.

    - Depending on the output of your WorkOnCaseFunction, your workflow may have ended by resolving the support case and closing the ticket, or escalating the ticket to the next tier of support. You can rerun the execution a few more times to observe this different behavior. This image shows an execution of the workflow where the support case was escalated, causing the workflow to exit with a Fail state.

    - In a real-world scenario, you might decide to continue working on the case until it’s resolved instead of failing out of your workflow. To do that, you could remove the Fail state and edit the Escalate Case Task in your state machine to loop back to the Work On Case state. No changes to your Lambda functions would be required. The functions we built for this tutorial are samples only, so we will move on to the next step of the tutorial.

# Congratulations, you created a serverless workflow with `AWS Lambda` and `Step Functions`

    Congratulations! You just created a serverless workflow using AWS Step Functions that invokes multiple AWS Lambda functions. Your workflow coordinated all of your functions according to the logic you defined, and passed data from one state to another, which meant you didn’t need to write that code into each individual function.

6. Clean up resources

   In this step, you will terminate your `AWS Step Functions` and `AWS Lambda` resources.

   - At the top of the AWS Step Functions console window, choose State machines.

   - In the State machines window, select CallCenterStateMachine and choose Delete. To confirm you want to delete the state machine, choose Delete state machine in the dialog box that appears. Your state machine will be deleted in a minute or two, after Step Functions has confirmed that any in-process executions have completed.

   - In the AWS Lambda console, on the Functions screen, select each of the functions you created for this tutorial and then select Actions and then Delete. Confirm the deletion by selecting Delete again.

   - Lastly, you will delete your IAM roles. In the IAM console in the Roles section, enter step into the search, select step_functions_basic_execution, and choose Delete.

   - You can now sign out of the `AWS Management console`.
