# AWS-Playbooks

## Overview

This repository contains Ansible playbooks designed to automate various AWS tasks. These playbooks streamline the provisioning, management, and deployment of AWS resources, ensuring consistent and efficient operations.

## Playbooks Included

- **provision_ec2.yml**: Automates the provisioning of EC2 instances with specified configurations.

## Prerequisites

Before using these playbooks, ensure the following:

- **Ansible**: Installed on your local machine.
- **AWS Credentials**: Configured with the necessary permissions to execute the tasks defined in the playbooks.
- **Python**: Version 3.7 or higher is recommended.
- **Jenkins**: Set up and configured for executing the provided pipeline script.
- **DeployEase**: Application to manage and update the `vars.yml` file dynamically with required parameters.

## Integration with DeployEase

The `vars.yml` file required by these playbooks is dynamically updated using the [DeployEase](https://github.com/HasaraDA/DeployEase) application. DeployEase simplifies the management of application deployment configurations and ensures that the latest values are available for Ansible playbooks.

### Updating `vars.yml` with DeployEase

1. Clone the DeployEase repository:

   ```bash
   git clone https://github.com/HasaraDA/DeployEase.git
   ```

2. Follow the instructions in the DeployEase repository to set up and update `vars.yml`.

3. Once updated, copy or link the updated `vars.yml` file to the `AWS-Playbooks` repository directory.

## Automated Workflow with Webhooks

### Webhook Integration

The entire process from provisioning a server to sending the notification email is automated via webhooks. As soon as a server is requested through the **DeployEase** application, the following sequence occurs:

1. **DeployEase** updates the `vars.yml` file with the new server details.
2. The webhook set up within the **DeployEase** application automatically triggers the **Jenkins pipeline**.
3. Jenkins then executes the playbook to provision the EC2 instance using the updated `vars.yml`.
4. After provisioning, Jenkins sends an email to the user with the EC2 instance details (server name, IPs, instance type, etc.).

This process ensures that all tasks are handled seamlessly, allowing users to receive timely notifications about their requested servers.

### Running the Playbook Directly

1. **Clone the Repository**:

   ```bash
   git clone https://github.com/HasaraDA/AWS-Playbooks.git
   ```

2. **Navigate to the Directory**:

   ```bash
   cd AWS-Playbooks
   ```

3. **Ensure `vars.yml` is Updated**:

   Use the DeployEase application to update the `vars.yml` file.

4. **Run the Playbook**:

   Execute the playbook using the following command:

   ```bash
   ansible-playbook -i hosts provision_ec2.yml
   ```

### Running via Jenkins Pipeline

This playbook can also be executed using the Jenkins pipeline script provided below:

```groovy
pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID = credentials('aws-key-id')  // Reference AWS credentials stored in Jenkins
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        AWS_DEFAULT_REGION = 'ap-southeast-1'  // Your AWS region
    }

    stages {
        stage('GIT Checkout') {
            steps {
                // Pull the playbooks from your GitHub repository
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/HasaraDA/AWS-Playbooks'

                // List the files to verify that the playbook is checked out
                sh 'ls -l'
            }
        }

        stage('Run EC2 Provision Playbook') {
            steps {
                script {
                    def provisionOutput = sh(script: 'ansible-playbook provision_ec2.yml', returnStdout: true).trim()
                    echo "Provision Output: ${provisionOutput}"
                }
            }
        }

        stage('Read vars.yaml and Send Email') {
            steps {
                script {
                    try {
                        def vars = readYaml file: 'vars.yml'
                        def userEmail = vars.user_email

                        def emailBody = """
                            Server Name: ${env.serverName}
                            Private IP: ${env.privateIp}
                            Public IP: ${env.publicIp}
                            Instance Type: ${env.instanceType}
                            Region: ${env.region}
                            Image: ${env.image}
                            KeyPair: ${env.keyPair}
                        """

                        emailext (
                            subject: "EC2 Instance Provisioning Details: ${env.serverName}",
                            body: """<html>
                                        <body>
                                            ${emailBody.replace('\n', '<br>')}
                                        </body>
                                    </html>""",
                            to: userEmail,
                            from: 'hasara.assal@gmail.com',
                            replyTo: 'hasara.assal@gmail.com',
                            mimeType: 'text/html'
                        )
                    } catch (Exception e) {
                        echo "Failed to send email. Error: ${e.message}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'EC2 instance provisioned successfully and email sent.'
        }
        failure {
            echo 'There was an error provisioning the EC2 instance.'
        }
    }
}
```

## Customization

These playbooks are intended as templates and may require customization to fit your specific needs. Modify the tasks and variables as necessary to align with your AWS environment and operational requirements.

## Contributing

Contributions are welcome! Feel free to submit issues or pull requests to enhance the functionality of these playbooks.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Notes

- Ensure that your Jenkins environment is configured with the required AWS credentials and GitHub access tokens.
- Always test playbooks in a controlled environment before deploying them in production.