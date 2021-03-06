## Lab -2 - Integrate Comprehend Medical

This section of the lab will focus on integrating Comprehend Medical with the FHIR interface. We will first load data into the FHIR repository that we created in Lab 1. The data is generated using Synthea. We will validate that the data is loaded by retrieving the patient based on the patient id. Next, we make a call to get the conditions for the patient. At this stage only the conditions that were part of the test file should be present. We then use a HL7 MDM file which has unstructured text in OBX segments as a data source, extarct conditions from this text, map it to FHIR Condition resource and load it to the repository. The patient should now have the additional Condition's loaded.


## Architecture & Use cases

![CM Integration](images/part-2-image-1.png)

1. Run the following command to go back to the environment directory.

    ```
    cd ~/environment/
    ```

1. Run the following command to download code for FHIR Comprehend Medical integration. The git credentials to use are as below:

    ```
    git clone https://github.com/aws-samples/amazon-comprehend-medical-fhir-integration.git
    ```

## Load data to FHIR repository

We will now load a patient record to an existing FHIR repository. The test file used here was generated using Synthea. Synthea is a synthetic patient generator that models the medical history of synthetic patients. It is widely used in the healthcare industry to output high-quality synthetic, realistic but not real, patient data and associated health records covering every aspect of healthcare. More details about Synthea can be found [here](https://github.com/synthetichealth/synthea/wiki).

1. Go to test-data folder by running following command.
    ```
    cd amazon-comprehend-medical-fhir-integration/test-data/master-patient-data/
    ```
## Extract key patient info 

1. Run the following command to extract the required info for the workshop.
    ```
    python ~/environment/amazon-comprehend-medical-fhir-integration/resources/extract-patient-info.py \
    ~/environment/amazon-comprehend-medical-fhir-integration/test-data/master-patient-data/patient-bundle.json
    ```

The output would be similar to the screenshot below. 


![CM Integration](images/part-2-image-2.png)

The goal of this step is to show you some of the key elements of the patient record that will be loaded into the FHIR repository.
**Copy the output to a notepad or editor of your choice to validate in later steps. The patient id will be used in later steps to retrieve the patient record.**

1. Change the terminal directory by running following command:

    ```
    cd ~/environment/amazon-comprehend-medical-fhir-integration/resources/
    ```

1. Run the following commands to set the executable permissions and then execute the script to load the master patient record in the FHIR repository. Please give it a few minutes for the data load to complete.

    ```
    chmod u+x post-test-data.sh
    ./post-test-data.sh
    ```

The output should be similar to below:
![CM Integration](images/part-2-image-3.png)

1. Let us validate the data by running following command to retrieve a patient’s data. Replace the patient id with the value extracted in previous step. Click [here](#extract-key-patient-info) if you need to extract patient info again.

    ```
    curl -H "Accept: application/fhir+json" \
         -H "Authorization:$ID_TOKEN" "$API_EDNPOINT"Patient/<<PATIENT_ID>> | jq
    ```

1. Get the conditions for the patient by running the following command

    ```
    curl -H "Accept: application/fhir+json" \
         -H "Authorization:$ID_TOKEN" "$API_EDNPOINT"Condition?patient-ref-id=<<PATIENT_ID>> | jq
    ```


**Make a note of the number of conditions for the patient. We will be adding additional conditions extracted from the clinical notes. In this lab we used a patient with two conditions. However, you can confirm it by looking at the "total" tag value in the json output.**

## Building  amazon-comprehend-medical-fhir-integration project 

1. Run the following from terminal to switch to the comprehend medical directory:

    ```
    cd /home/ec2-user/environment/amazon-comprehend-medical-fhir-integration/
    ```

1. Next, build the project by running the following mavan command. 

    ```
    mvn clean install
    ```

The build process will create a single jar file. It has different classes that are deployed as lambda functions to process hl7 and FHIR resource files, call comprehend medical api's and make API calls to an existing FHIR repository.


## Deploy the project code ##

We will now deploy the workshop code using a SAM(Serverless Access Model) template. More details about SAM can be found [here](https://docs.aws.amazon.com/serverless-application-model/index.html).

1. Go to **resources** folder under amazon-comprehend-medical-fhir-integration.  Change directory to resources folder.
    ```
    cd ~/environment/amazon-comprehend-medical-fhir-integration/resources
    ```
1. Run the following command to package the code and other artifacts. Use the same bucket name from part-1 of the lab.

    ```
    aws cloudformation package --template-file FHIR-CM-Integration.yaml \
        --output-template-file serverless-output.yaml \
        --s3-bucket $PACKAGE_BUCKET_NAME
    ```
Feel free to explore the template by going to FHIR-CM-Integration.yaml file under the resources folder. It will have the resource defintions for the services that have been used as part of this project. The main services that orchestrates the flow are AWS Step functions, AWS Lambda functions and Amazon Comprehend Medical.

1. Run the following command to deploy the SAM template.

    ```
    aws cloudformation deploy --template-file \
        ~/environment/amazon-comprehend-medical-fhir-integration/resources/serverless-output.yaml \
        --stack-name fhir-cm-integ \
        --capabilities CAPABILITY_IAM \
        --parameter-overrides CognitoSecretName=fhir-cm-secret-store \
        FHIRAPIEndpoint=$API_EDNPOINT ClientId=$CLIENT_ID
    ```
## Get data input bucket
1. Get DATA_INPUT_BUCKET by running the following command. Save this value for use in later steps. This is the source bucket to upload HL7 and FHIR data files.

    ```
    DATA_INPUT_BUCKET="$(aws cloudformation describe-stacks \
        --stack-name fhir-cm-integ \
        --query 'Stacks[].Outputs[?OutputKey==`BucketName`][OutputValue]' \
        --output text)"
    export DATA_INPUT_BUCKET="${DATA_INPUT_BUCKET}"    
    ```

## HL7 Message Flow

In this part of the lab, we will upload a HL7 file which has OBX segment containing clinical notes. The deployed project would extract the notes from the HL7 message, lookup the patient from FHIR repository and enrich the record with the medical conditions extracted from the clinical notes in HL7. Typically customers need to build some manual workflows or custom rule engines to extract the clinical entities from such segments. In this use case, we have automated the pipeline by ingesting the HL7 messages to S3 and integrating it with Amazon Comprehend Medical using AWS step functions.

1. Go to the resources/test-data folder under FHIR-CM-Integration by running following command from terminal.

    ```
    cd ~/environment/amazon-comprehend-medical-fhir-integration/test-data
    ```

1. Open the mdm.txt file under test-data directory. It is populated based on test data that was used as part of this lab. It represents the HL7 message for the same patient loaded in FHIR repository.

1. Upload the file to s3 folder using the command below. It would trigger the flow to extract the data from OBX segments, run it through Comprehend Medical, extract the Condition resources and enrich data in FHIR repository.

    ```
    aws s3 cp mdm.txt s3://"$DATA_INPUT_BUCKET"/input/hl7/mdm.txt
    ```

The above command should trigger the step function. Click on AWS Cloud9 link and select Go To Your Dashboard. It will take you back to your environments.  
![FHIR Server](images/part-1-image-15.png)

You can search for services in that page.
![FHIR Server](images/part-1-image-16.png)

Search for step functions service on the search bar. You will see a listing of the step functions that are currently running or completed. 
![SFN Listing](images/part-2-image-5.png)

Click on the step function that is deployed as part of the workshop. It would show the progress of the various steps.
![CM Integration](images/part-2-image-4.png)
1. Run the below command again to lookup the conditions for the patient. You will need to replace the patient id with the value captured in the previous step of extracting patient info.

    ```
    curl -H "Accept: application/fhir+json" -H "Authorization:$ID_TOKEN" \
    "$API_EDNPOINT"Condition?patient-ref-id=<<PATIENT_ID>> | jq

    ```

    **Note: Run the below two commands ONLY if you get an expired token error. It will reset the auth token and you can try the curl request again after that.**

    >```
    >NEW_ID_TOKEN="$(python ~/environment/amazon-comprehend-medical-fhir-integration/resources/init-auth.py $CLIENT_ID)" 
    >```

    >```
    >export ID_TOKEN="${NEW_ID_TOKEN}"
    >```


    Look for the conditions again. It should show the additional conditions that were extracted from the hl7 message. It will now have 5 conditions loaded for the patiet. This completes the second part of the lab.

## Optional - FHIR Message flow
The FHIR message flow demonstrates the scenario where an existing FHIR resource like DocumentReference has clinical notes embedded in one of the tags. In this example, we have embedded notes in the data tag of the content element. The notes are sent as a base 64 encoded string. You can decode the data by pasting it [here](https://www.base64decode.org/).

1. Run the following command to change directory to resources directory.

    ```
    cd /home/ec2-user/environment/amazon-comprehend-medical-fhir-integration/
    ```

1. On the left navigation pane look for the file FHIR-DocRef.json under amazon-comprehend-medical-fhir-integration/test-data. Open it in the editor to make some changes.
1. Look for the tag *REPLACE_WITH_PATIENT_ID* and replace it with the patient id used in scenario 1. Make sure to save the file.The purpose of the change is to extract condition resources from clinical notes embedded in the data tag of the resource and load it as Condition resources for the patient.
1. Run the following command to upload the file to trigger the workflow:

    ```
    aws s3 cp test-data/FHIR-DocRef.json s3://"$DATA_INPUT_BUCKET"/input/fhir/FHIR-DocRef.json
    ```

The above command should trigger the same step function from lab 1. You can monitor the progress of the step function by logging in to the console and going to the step functions service. You will see a listing of the step functions that are currently running or completed. 
![SFN Listing](images/part-2-image-5.png)Click on the step function that is deployed as part of the workshop. It would show the progress of the various steps.![CM Integration](images/part-2-image-4.png)

1. Once the step functions is completed, run the following command to get the Conditions for the patient. It should now be updated with the new conditions that were loaded.
    ```
    curl -H "Accept: application/fhir+json" -H "Authorization:$ID_TOKEN" \
    "$API_EDNPOINT"Condition?patient-ref-id=<<PATIENT_ID>> | jq
    ```
It should now show the additional Conditions that were loaded from the notes in the DocumentReference resource.


 

## Cleanup 

Once you have completed the workshop, run the following commands to delete the cloudformation stacks. It will delete all the resources (except S3 buckets) that were created as part of the workshop.
```
aws cloudformation delete-stack \
    --stack-name fhir-cm-integ

aws cloudformation delete-stack \
    --stack-name aws-fhir-interface

```

You can delete the S3 buckets manually by going to the S3 console, selecting the bucket names and click on delete. It will ask you to confirm by entering the name of the bucket. Also, delete the cloud9 environment that was created to run the lab.

Congratulations! You have now completed the full lab.

Click [here](lab1.md) to go to previous lab.