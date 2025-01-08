## Lambda code to terminate the unused instances on all regions in DEV account

1. Account - DEV
2. Region  - us-east-1
3. Service - Lambda
4. Name  - Clear- Instances

### Task Description

The task is about to terminate all the instances which are running in the DEV enviroment after 8.00 pm. 
This created lamda will terminate all the instances across all the regions in the dev account except TAG name of the instances starts or ends with AURA

# Login aws -> lambda-> create function -> 
## Step:1
create the lambda function and select the existing role for the lambda function 
![image](https://github.com/user-attachments/assets/2b3861b4-b0df-4bd6-abf8-ce608894f350)

![image (6)](https://github.com/user-attachments/assets/47dddb29-93cb-4883-be64-4e65b3223df1)

## Step: 2
Once the function is created. Copy and paste the below python code in the code section and deploy the lambda function

```
import boto3

# Initialize AWS SDK (Boto3) client for EC2
ec2_client = boto3.client('ec2')

def terminate_all_running_instances_in_all_regions():
    # Get a list of all available regions
    regions_response = ec2_client.describe_regions()
    regions = [region['RegionName'] for region in regions_response['Regions']]

    print(f"Regions to check: {regions}")  # Debug: Log all regions

    for region in regions:
        print(f"Checking region: {region}")
        regional_ec2_client = boto3.client('ec2', region_name=region)
        
        # Get all running EC2 instances in the current region
        response = regional_ec2_client.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
        instances_to_terminate = []

        for reservation in response['Reservations']:
            for instance in reservation['Instances']:
                # Check if the instance has a Name tag
                name = None
                for tag in instance.get('Tags', []):
                    if tag['Key'] == 'Name':
                        name = tag['Value']
                        break
                
                # If the instance name contains "Aura", skip it
                if name and "Aura" in name:
                    print(f"Skipping instance {instance['InstanceId']} with name {name} in region {region}")
                    continue
                
                # Check if termination protection is enabled
                instance_id = instance['InstanceId']
                protection_response = regional_ec2_client.describe_instance_attribute(
                    InstanceId=instance_id,
                    Attribute='disableApiTermination'
                )
                termination_protection_enabled = protection_response['DisableApiTermination']['Value']
                
                if termination_protection_enabled:
                    print(f"Skipping instance {instance_id} as termination protection is enabled in region {region}")
                    continue
                
                # Otherwise, mark it for termination
                instances_to_terminate.append(instance_id)
        
        if not instances_to_terminate:
            print(f"No instances to terminate in region: {region}")
            continue
        
        # Terminate the marked instances
        print(f"Terminating instances in {region}: {instances_to_terminate}")
        regional_ec2_client.terminate_instances(InstanceIds=instances_to_terminate)

def lambda_handler(event, context):
    terminate_all_running_instances_in_all_regions()
    
    return {
        'statusCode': 200,
        'body': "Terminated all running instances except those containing 'Aura' in their name or have termination protection enabled."
    }
```

![image](https://github.com/user-attachments/assets/1e6abafc-7a26-4ca4-ba0c-0712d38ba21f)

## Step: 3

> [!NOTE]
> you can create a new rule or use the existing rule named clear-instance-rule as specified in the screen shot.

```
cron(30 1 * * ? *)
```

![image](https://github.com/user-attachments/assets/d776bff7-22d7-40d3-926f-192c32b671d2)

## Step: 4

Apply the test button as shown in the step 2 screen shot to see the instant change. Verify the changes that instances are cleared/terminated.




