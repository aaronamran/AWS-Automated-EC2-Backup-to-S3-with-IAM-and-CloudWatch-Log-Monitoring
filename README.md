# Automated EC2 Backup to S3 with IAM and CloudWatch Log Monitoring


1. [Configuring IAM Role-Based Access](#configuring-iam-role-based-access)
2. [Setting up EC2, AWS CLI and CloudWatch Agent](#setting-up-ec2-aws-cli-and-cloudwatch-agent)
3. [Creating S3 Bucket and Scripting the Backup Process](#creating-s3-bucket-and-scripting-the-backup-process)
4. [CloudWatch Logging and Testing Backup Process](#cloudwatch-logging-and-testing-backup-process)


## Configuring IAM Role-Based Access
- Login to AWS and navigate to IAM > Roles. Click Create role
  ![image](https://github.com/user-attachments/assets/df84728f-e282-4dbe-bc41-34daf6f6d6d8) <br />

- Select AWS service as the trusted entity type and EC2 as the service or use case
  ![image](https://github.com/user-attachments/assets/906f617b-7d30-4cb0-85de-3ca18215a581) <br />

- Attach the following permission policies: `AmazonS3FullAccess` and `CloudWatchLogsFullAccess`
  ![image](https://github.com/user-attachments/assets/ea78fa8d-647e-4518-a450-670a21676670) <br />

- Give a name to it like `EC2S3BackupRole` and create the role
  ![image](https://github.com/user-attachments/assets/bd65ee54-feac-4d14-bdf2-76bf3a9328b1) <br />

- The role is successfully created
  ![image](https://github.com/user-attachments/assets/32b77ecd-aac2-42fe-890c-2906e3afb262) <br />



## Setting up EC2, AWS CLI and CloudWatch Agent
- Navigate to EC2 Instance and setup the instance. Give a name like `TempEC2Instance`. The OS image used is Amazon Linux
  ![image](https://github.com/user-attachments/assets/ab94f0fe-491f-4ea5-9163-a63e12262f54) <br />

- Leave the Amazon Linux 2023 AMI and instance type of t2.micro as Free tier eligible
  ![image](https://github.com/user-attachments/assets/002a47ca-bc71-4e08-acd9-4e404b42844c) <br />

- Either create or use an existing key pair for login purposes
  ![image](https://github.com/user-attachments/assets/d4d7c6c6-0605-4c6a-99d5-a4151802c9f9) <br />

- In terms of Inbound Security Group Rules, allow SSH only from your IP. By default, AWS allows all outbound traffic
  ![image](https://github.com/user-attachments/assets/b2a2dfec-69e2-4210-a18d-6b408b950a93) <br />

- Leave the storage configuration as default and launch the instance
  ![image](https://github.com/user-attachments/assets/dd1d2673-dc23-4163-abf0-459a7fb1183f) <br />

- Modify the IAM role of the running instance
  ![image](https://github.com/user-attachments/assets/f3fb1bdd-5be1-430d-9ad7-9344f9baaa22) <br />

- Select the `EC2S3BackupRole`
  ![image](https://github.com/user-attachments/assets/0e9ad572-f88c-4863-b565-dc023da45af7) <br />

- To connect to the EC2 instance via a locally installed WSL Ubuntu, navigate to the instance summary. Then click Connect
  ![image](https://github.com/user-attachments/assets/04e87c0a-fe2a-4da3-9214-7a3490c8cf54) <br />

- It will show the steps on how to connect via SSH
  ![image](https://github.com/user-attachments/assets/b959ed95-e56e-4506-b019-c4524ebceb5d) <br />
  ![image](https://github.com/user-attachments/assets/3f7aee9c-6a1f-4799-a2f3-7bd10576a0c0) <br />

- Run each of the following commands below to install AWS CLI and CloudWatch Agent depending on which OS Image was chosen for the EC2 instance earlier
  ```
  # Amazon Linux 2
  sudo yum install -y awscli amazon-cloudwatch-agent
  
  # Ubuntu
  sudo apt update && sudo apt install -y awscli amazon-cloudwatch-agent
  ```
  ![image](https://github.com/user-attachments/assets/22de8cc0-421c-4b0a-b3d4-07f39977f144)



## Creating S3 Bucket and Scripting the Backup Process
- Via the SSH session, run the following command to create the S3 Bucket. Note that the bucket name must be globally unique
  ```
  aws s3 mb s3://my-ec2-backup-bucket-20250512
  ```
  ![image](https://github.com/user-attachments/assets/2be1f94e-c92e-450a-88dd-33131040fb6c) <br />

- First create a folder in the `/home/ec2-user/data` directory
  ```
  mkdir /home/ec2-user/data
  echo "This is a test backup file" > /home/ec2-user/data/test.txt
  ```
  ![image](https://github.com/user-attachments/assets/441572e5-c631-4140-aff1-5cd536671ad0)

- Create the backup script called `backup.sh`
  ```
  #!/bin/bash
  
  TIMESTAMP=$(date +%F-%H%M)
  BACKUP_SRC="/home/ec2-user/data"        # Change this to your source folder
  S3_BUCKET="s3://my-ec2-backup-bucket-20250512"
  LOGFILE="/var/log/backup.log"
  
  # Sync to S3
  echo "[$(date)] Starting backup..." >> $LOGFILE
  aws s3 sync $BACKUP_SRC $S3_BUCKET/backup-$TIMESTAMP >> $LOGFILE 2>&1
  echo "[$(date)] Backup complete." >> $LOGFILE
  ```
  ![image](https://github.com/user-attachments/assets/ceae9cbe-1654-4c20-9d8f-c9968326f48c)

- Make the shell script executable
  ```
  chmod +x backup.sh
  ```


## CloudWatch Logging and Testing Backup Process
- Create a CloudWatch agent config file
  ```
  sudo touch /opt/aws/amazon-cloudwatch-agent/etc/backup-logs-config.json
  ```
  Open the file with a text editor and paste the following json into the file
  ```
  {
    "logs": {
      "logs_collected": {
        "files": {
          "collect_list": [
            {
              "file_path": "/var/log/backup.log",
              "log_group_name": "EC2-Backup-Logs",
              "log_stream_name": "{instance_id}-backup",
              "timestamp_format": "%Y-%m-%d %H:%M:%S"
            }
          ]
        }
      }
    }
  }
  ```
  ![image](https://github.com/user-attachments/assets/e40094ad-2cb2-42e8-9cbb-ed2fce323101)


- Start the CloudWatch agent
  ```
  sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/backup-logs-config.json \
    -s
  ```
  ![image](https://github.com/user-attachments/assets/f3f57e30-c95f-4679-8be6-6dba516c7fac) <br />
  This will push `/var/log/backup.log` to CloudWatch Logs and group it under EC2-Backup-Logs

- Now test the script manually
  ```
  sudo ./backup.sh
  ```
  ![image](https://github.com/user-attachments/assets/b5e7d437-bce8-473e-b8f7-204170d19cee) <br />

  Then check if the S3 bucket has backup-YYYY-MM-DD-HHMM folder <br />
  ![image](https://github.com/user-attachments/assets/7788d513-0204-4692-8426-df7fa43c02ff) <br />
  
  Also check if the CloudWatch Logs contains new logs in EC2-Backup-Logs <br />
  ![image](https://github.com/user-attachments/assets/aab0ccf3-e20f-4928-8daf-401281e5dff2) <br />
  ![image](https://github.com/user-attachments/assets/68edf899-6934-484f-a32e-dd04cb562c96) <br />

- To add a Cron job to automate backing up data every hour, use
  ```
  crontab -e
  ```
  ![image](https://github.com/user-attachments/assets/8bda0cb9-5d48-4fd4-93d3-b67dd65b7eea) <br />
  If it is not found, install the cronie package (cron daemon)
  ```
  sudo yum install -y cronie
  ```
  ![image](https://github.com/user-attachments/assets/4e83379e-cf49-429e-a5c1-d800864551d0) <br />

  Start and enable the cron service
  ```
  sudo systemctl start crond
  sudo systemctl enable crond
  ```

  Now return to edit the crontab using nano text editor if vi or vim is too confusing
  ```
  export EDITOR=nano
  crontab -e
  ```

- And add this line to the bottom of the crontab file. This means run `backup.sh` every hour on the hour
  ```
  0 * * * * /home/ec2-user/backup.sh
  ```
  ![image](https://github.com/user-attachments/assets/c14af259-becc-485a-baff-57bc6e193089)


- Now the cron job will backup data every hour

  








