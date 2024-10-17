# Terraform을 이용한 AWS 인프라 구축 🚀

## 개요 📄

본 프로젝트는 Terraform을 사용하여 AWS 인프라를 자동으로 구축하는 과정을 다룹니다. 서브넷, 라우팅 테이블, 게이트웨이 등을 생성하고, 퍼블릭 및 프라이빗 라우트 테이블을 설정하여 효율적이고 안전한 네트워크 구성을 구현합니다.

## 팀원 👥

|<img src="https://avatars.githubusercontent.com/u/64997345?v=4" width="120" height="120"/>|<img src="https://avatars.githubusercontent.com/u/38968449?v=4" width="120" height="120"/>
|:-:|:-:|
|[@최영하](https://github.com/ChoiYoungha)|[@허예은](https://github.com/yyyeun)

## 학습 목표 🎯

- Terraform을 이용한 AWS 리소스 자동화 이해
- VPC, 서브넷, 라우팅 테이블, 게이트웨이 등의 네트워크 구성 요소 학습
- IAM 역할 및 정책 관리 방법 습득
- 인프라 코드 작성 및 관리 능력 향상

## 주요 개념 설명 💡

- **VPC (Virtual Private Cloud)**: AWS 클라우드 내에 논리적으로 격리된 네트워크를 생성.
- **서브넷 (Subnet)**: VPC 내에서 IP 주소 범위를 나누어 리소스를 배치.
- **라우팅 테이블 (Route Table)**: 트래픽의 흐름을 제어하는 규칙 집합.
- **인터넷 게이트웨이 (Internet Gateway)**: VPC와 인터넷 간의 트래픽을 허용.
- **NAT 게이트웨이 (NAT Gateway)**: 프라이빗 서브넷에서 인터넷으로의 아웃바운드 트래픽을 관리.
- **IAM 역할 및 정책 (IAM Role & Policy)**: AWS 리소스에 대한 접근 권한을 정의하고 관리.

## 사전 작업 🛠️

### 1. Terraform 설치

```bash
sudo su -

# HashiCorp GPG 키 추가
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# HashiCorp 리포지토리 추가
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 패키지 목록 업데이트 및 Terraform 설치
apt-get update && apt-get install terraform -y

# 설치 확인
terraform -version
```

### 2. AWS CLI 설치

```bash
# AWS CLI 설치
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 설치 확인
aws --version
```

## 작업 프로세스 📝

### 1. 역할과 정책 생성 후 연결 (`s3_iam_role.tf`)


```hcl
# IAM 역할 생성
resource "aws_iam_role" "s3_create_bucket_role" {
  name = "ce33-s3-create-bucket-role"
  
  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        }
      }
    ]
  })
}

# IAM 정책 정의 (S3에 대한 모든 권한 부여)
resource "aws_iam_policy" "s3_full_access_policy" {
  name        = "ce33-s3-full-access-policy"
  description = "Full access to S3 resources"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:*"  # 모든 S3 액세스 허용
        ]
        Resource = [
          "*"  # 모든 S3 리소스에 대한 권한
        ]
      }
    ]
  })
}

# IAM 역할에 정책 연결
resource "aws_iam_role_policy_attachment" "attach_s3_policy" {
  role       = aws_iam_role.s3_create_bucket_role.name
  policy_arn = aws_iam_policy.s3_full_access_policy.arn
}
```

### 2. VPC 생성 (`create_vpc.tf`)


```hcl
variable "vpc-cidr-block" {
  description = "VPC의 CIDR 블록"
  type        = string
  default     = "10.0.0.0/16" # 원하는 기본 CIDR 블록
}

resource "aws_vpc" "vpc" {
  cidr_block = var.vpc-cidr-block
  
  tags = {
    Name = "ce33-terraform-vpc"
  }
}
```

### 3. Web 서브넷 생성 (`web-subnets.tf`)


```hcl
# 첫 번째 서브넷의 CIDR 블록 변수 선언
variable "web-subnet1-cidr" {
  description = "첫 번째 서브넷의 CIDR 블록"
  type        = string
  default     = "10.0.1.0/24" # 서브넷1의 기본 CIDR 블록
}

# 두 번째 서브넷의 CIDR 블록 변수 선언
variable "web-subnet2-cidr" {
  description = "두 번째 서브넷의 CIDR 블록"
  type        = string
  default     = "10.0.2.0/24" # 서브넷2의 기본 CIDR 블록
}

# 첫 번째 서브넷 (web-subnet1) 생성
resource "aws_subnet" "web-subnet1" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = var.web-subnet1-cidr
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true

  tags = {
    Name = "ce33-terraform-subnet1"
  }
}

# 두 번째 서브넷 (web-subnet2) 생성
resource "aws_subnet" "web-subnet2" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = var.web-subnet2-cidr
  availability_zone       = "ap-northeast-2b"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "ce33-terraform-subnet2"
  }
}
```

### 4. 프라이빗 Web 서브넷 생성 (`app-subnets.tf`)


```hcl
# CIDR 블록 변수 선언
variable "app-subnet1-cidr" {
  description = "CIDR block for the first application subnet"
  type        = string
  default     = "10.0.11.0/24"  # 예시 CIDR 블록
}

variable "app-subnet2-cidr" {
  description = "CIDR block for the second application subnet"
  type        = string
  default     = "10.0.21.0/24"  # 예시 CIDR 블록
}

# 첫 번째 애플리케이션 서브넷 (app-subnet1) 생성
resource "aws_subnet" "app-subnet1" {
  vpc_id                  = aws_vpc.vpc.id  
  cidr_block              = var.app-subnet1-cidr
  availability_zone       = "ap-northeast-2a" 
  map_public_ip_on_launch = false

  tags = {
    Name = "ce33-terraform-app_subnet1"
  }
}

# 두 번째 애플리케이션 서브넷 (app-subnet2) 생성
resource "aws_subnet" "app-subnet2" {
  vpc_id                  = aws_vpc.vpc.id  
  cidr_block              = var.app-subnet2-cidr  
  availability_zone       = "ap-northeast-2b"    
  map_public_ip_on_launch = false

  tags = {
    Name = "ce33-terraform-app_subnet2"
  }
}
```

### 5. 인터넷 게이트웨이 생성 (`internet-gw.tf`)

```hcl
# 인터넷 게이트웨이 생성
resource "aws_internet_gateway" "internet-gw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "ce33-terraform-igw"  # 인터넷 게이트웨이 이름
  }
}
```

### 6. NAT 생성을 위한 Elastic IP 생성 (`eip.tf`)


```hcl
# Elastic IP 생성
resource "aws_eip" "eip" {
  domain = "vpc"

  tags = {
    Name = "ce33-terraform-eip"  # Elastic IP의 이름
  }
}
```

### 7. NAT 게이트웨이 생성 (`nat-gw.tf`)


```hcl
# NAT 게이트웨이 생성
resource "aws_nat_gateway" "nat-gw" {
  allocation_id     = aws_eip.eip.id
  connectivity_type = "public"
  subnet_id         = aws_subnet.web-subnet1.id

  tags = {
    Name = "ce33-terraform-ngw"  # NAT 게이트웨이의 이름
  }

  depends_on = [aws_internet_gateway.internet-gw]
}
```

### 8. 퍼블릭 라우트 테이블 생성 (`public-rt.tf`)


```hcl
# 퍼블릭 라우트 테이블 생성
resource "aws_route_table" "public-route-table" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"  # 모든 IP 주소에 대한 트래픽
    gateway_id = aws_internet_gateway.internet-gw.id  # 인터넷 게이트웨이를 통해 라우팅
  }

  tags = {
    Name = "ce33-terraform-public-rt"  # 퍼블릭 라우트 테이블의 이름
  }
}

# 첫 번째 퍼블릭 라우트 테이블과 서브넷의 연결
resource "aws_route_table_association" "pub-rt-association-1" {
  subnet_id      = aws_subnet.web-subnet1.id
  route_table_id = aws_route_table.public-route-table.id
}

# 두 번째 퍼블릭 라우트 테이블과 서브넷의 연결
resource "aws_route_table_association" "pub-rt-association-2" {
  subnet_id      = aws_subnet.web-subnet2.id
  route_table_id = aws_route_table.public-route-table.id
}
```

### 9. 프라이빗 라우트 테이블 생성 (`private-rt.tf`)


```hcl
# 프라이빗 라우트 테이블 생성
resource "aws_route_table" "private-route-table" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block     = "0.0.0.0/0"  # 모든 IP 주소에 대한 트래픽
    nat_gateway_id = aws_nat_gateway.nat-gw.id  # NAT Gateway로 수정
  }

  tags = {
    Name = "ce33-terraform-private-rt"  # 프라이빗 라우트 테이블의 이름
  }
}

# 첫 번째 프라이빗 라우트 테이블과 서브넷의 연결
resource "aws_route_table_association" "pri-rt-association-1" {
  subnet_id      = aws_subnet.app-subnet1.id
  route_table_id = aws_route_table.private-route-table.id
}

# 두 번째 프라이빗 라우트 테이블과 서브넷의 연결
resource "aws_route_table_association" "pri-rt-association-2" {
  subnet_id      = aws_subnet.app-subnet2.id
  route_table_id = aws_route_table.private-route-table.id
}
```

## 공통 명령어 📟

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

## 결과 🎉
![2024-10-17 12 09 44](https://github.com/user-attachments/assets/2cc614fb-bdc7-4d16-a8a7-3c0953af5b9c)


## 자원 삭제 🗑️

테스트로 생성한 AWS 리소스를 지우고 비용을 절약합니다.
```bash
terraform destroy -auto-approve
```

## 결론 및 고찰 📝

Terraform을 활용한 AWS 인프라 자동화의 기본 개념과 실무 적용 방법을 학습하였습니다. 코드 기반의 인프라 관리는 반복 가능하고, 일관된 환경 구축을 가능하게 하여 운영 효율성을 크게 향상시킵니다. 향후 보안 강화, 모니터링 추가 등 추가적인 개선을 통해 더욱 견고한 클라우드 인프라를 구축할 계획입니다.

---