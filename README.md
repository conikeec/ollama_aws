# Deploy Ollama on EC2

This setup is ideal for leveraging open-sourced local Large Language Model (LLM) AI capabilities within the cloud, ensuring data privacy, customizability, regulatory compliance, and cost-effective AI solutions

# LLM: The Evolution from Traditional Models
The journey from traditional LLMs to llama.cpp marks a significant shift. Traditional models required high-end GPUs and extensive technical expertise, making them inaccessible to many.
llama.cpp, however, democratizes AI by optimizing for CPUs and simplifying implementation, broadening the scope for individual developers and smaller teams.

# Why Ollama?
Ollama stands out for enabling local access to large language models like Llama 2 and Code Llama, ensuring data privacy and security. It supports GPU acceleration on both macOS and Linux and provides programming libraries in Python and JavaScript. Under the MIT License, Ollama offers an accessible and flexible Gen AI solution.

https://github.com/ollama/ollama

# Step-by-Step Guide to Setting Up Ollama on EC2

![alt text](<ollama_aws.png>)

## Step 1: Initialize Your EC2 Instance

The below configuration is for a GPU enabled EC2 instance, however it can be done on a CPU only instance as well. For CPU based instances we can skip the NVIDIA driver setup.

Configure an Amazon Linux 2 EC2 instance:

Instance Type: g4dn.xlarge (~ $390 per month for the below configuration). Reference
- vCPU: 4
- RAM: 128GB
- GPU: 1 (VRAM: 16GB)
- EBS: 100GB (gp3)
- OS: Amazon Linux 2
- SSH Key: Required for PuTTY login

```hcl
provider "aws" {
  region = "us-west-2"  # Replace with your desired region
}

resource "aws_instance" "ollama" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 AMI
  instance_type = "g4dn.xlarge"
  key_name      = "your-key-pair"  # Replace with your key pair name
  
  root_block_device {
    volume_size = 100
    volume_type = "gp3"
  }
  
  tags = {
    Name = "ollama-server"
  }
}
```

# Step 2: Create an Instance Role with S3 access

```hcl
resource "aws_iam_role" "ollama_role" {
  name = "ollama-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ollama_s3_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
  role       = aws_iam_role.ollama_role.name
}

resource "aws_iam_instance_profile" "ollama_profile" {
  name = "ollama-profile"
  role = aws_iam_role.ollama_role.name
}

resource "aws_instance" "ollama" {
  # ... other configuration ...
  iam_instance_profile = aws_iam_instance_profile.ollama_profile.name
}
```

# Step 3: Install NVIDIA GRID Drivers

```hcl
resource "aws_instance" "ollama" {
  # ... other configuration ...
  
  user_data = <<-EOF
              #!/bin/bash
              sudo yum update -y
              sudo yum install -y gcc kernel-devel-$(uname -r)
              aws s3 cp --recursive s3://ec2-linux-nvidia-drivers/latest/ .
              chmod +x NVIDIA-Linux-x86_64*.run
              mkdir /home/ec2-user/tmp
              chmod -R 777 tmp
              export TMPDIR=/home/ec2-user/tmp
              CC=/usr/bin/gcc10-cc ./NVIDIA-Linux-x86_64*.run --tmpdir=$TMPDIR
              sudo touch /etc/modprobe.d/nvidia.conf
              echo "options nvidia NVreg_EnableGpuFirmware=0" | sudo tee --append /etc/modprobe.d/nvidia.conf
              EOF
}
```

# Step 4: Set Up Docker Engine

```hcl
resource "aws_instance" "ollama" {
  # ... other configuration ...
  
  user_data = <<-EOF
              #!/bin/bash
              # ... NVIDIA driver installation ...
              sudo yum install -y docker
              sudo usermod -a -G docker ec2-user
              sudo systemctl enable docker.service
              sudo systemctl start docker.service
              EOF
}
```

# Step 5 : Install NVIDIA Drivers for Docker

```hcl
resource "aws_instance" "ollama" {
  # ... other configuration ...
  
  user_data = <<-EOF
              #!/bin/bash
              # ... NVIDIA driver installation ...
              # ... Docker installation ...
              curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
                sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
              sudo yum install -y nvidia-container-toolkit
              EOF
}
```

# Step 6: Install Ollama Server Docker Container

```hcl
resource "aws_instance" "ollama" {
  # ... other configuration ...
  
  user_data = <<-EOF
              #!/bin/bash
              # ... NVIDIA driver installation ...
              # ... Docker installation ...
              # ... NVIDIA drivers for Docker installation ...
              # ... Configure Docker for NVIDIA drivers ...
              docker run -d --gpus=all -v ollama:/root/.ollama -p 11434:11434 --name ollama --restart always ollama/ollama
              EOF
}
```

# Step 7: Download Required LLM Models of your choice

```hcl
resource "aws_instance" "ollama" {
  # ... other configuration ...
  
  user_data = <<-EOF
              #!/bin/bash
              # ... NVIDIA driver installation ...
              # ... Docker installation ...
              # ... NVIDIA drivers for Docker installation ...
              # ... Configure Docker for NVIDIA drivers ...
              # ... Install Ollama Server Docker Container ...
              docker exec -it ollama ollama pull deepseek-llm
              docker exec -it ollama ollama pull llama2
              docker exec -it ollama ollama pull deepseek-coder:6.7b
              docker exec -it ollama ollama pull codellama:7b
              EOF
}
```


# Step 8: Install Ollama Web UI Container

```hcl
resource "aws_instance" "ollama" {
  # ... other configuration ...
  
  user_data = <<-EOF
              #!/bin/bash
              # ... NVIDIA driver installation ...
              # ... Docker installation ...
              # ... NVIDIA drivers for Docker installation ...
              # ... Configure Docker for NVIDIA drivers ...
              # ... Install Ollama Server Docker Container ...
              # ... Download Required LLM Models of your choice ...
              docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v ollama-webui:/app/backend/data --name ollama-webui --restart always ghcr.io/ollama-webui/ollama-webui:main
              EOF
}
```

# Step 12: Access the Ollama Web UI

You can access the Ollama Web UI using the private IP of the EC2 instance at http://<private-ip>:3000.
NOTE: Ensure that the URI is not exposed via public-ip (configure secure access)

### Test and verify accessibility

# Summary : Here is the complete script

```hcl
provider "aws" {
  region = "us-west-2"  # Replace with your desired region
}

resource "aws_iam_role" "ollama_role" {
  name = "ollama-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ollama_s3_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
  role       = aws_iam_role.ollama_role.name
}

resource "aws_iam_instance_profile" "ollama_profile" {
  name = "ollama-profile"
  role = aws_iam_role.ollama_role.name
}

resource "aws_instance" "ollama" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 AMI
  instance_type = "g4dn.xlarge"
  key_name      = "your-key-pair"  # Replace with your key pair name
  
  root_block_device {
    volume_size = 100
    volume_type = "gp3"
  }
  
  iam_instance_profile = aws_iam_instance_profile.ollama_profile.name
  
  user_data = <<-EOF
              #!/bin/bash
              sudo yum update -y
              sudo yum install -y gcc kernel-devel-$(uname -r)
              aws s3 cp --recursive s3://ec2-linux-nvidia-drivers/latest/ .
              chmod +x NVIDIA-Linux-x86_64*.run
              mkdir /home/ec2-user/tmp
              chmod -R 777 tmp
              export TMPDIR=/home/ec2-user/tmp
              CC=/usr/bin/gcc10-cc ./NVIDIA-Linux-x86_64*.run --tmpdir=$TMPDIR
              sudo touch /etc/modprobe.d/nvidia.conf
              echo "options nvidia NVreg_EnableGpuFirmware=0" | sudo tee --append /etc/modprobe.d/nvidia.conf
              sudo yum install -y docker
              sudo usermod -a -G docker ec2-user
              sudo systemctl enable docker.service
              sudo systemctl start docker.service
              curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
                sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
              sudo yum install -y nvidia-container-toolkit
              sudo nvidia-ctk runtime configure --runtime=docker
              sudo systemctl restart docker
              docker run -d --gpus=all -v ollama:/root/.ollama -p 11434:11434 --name ollama --restart always ollama/ollama
              docker exec -it ollama ollama pull deepseek-llm
              docker exec -it ollama ollama pull llama2
              docker exec -it ollama ollama pull deepseek-coder:6.7b
              docker exec -it ollama ollama pull codellama:7b
              docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v ollama-webui:/app/backend/data --name ollama-webui --restart always ghcr.io/ollama-webui/ollama-webui:main
              EOF
  
  tags = {
    Name = "ollama-server"
  }
}
```