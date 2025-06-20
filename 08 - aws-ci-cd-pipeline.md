## Objective:
### Set up a CI/CD pipeline using:
- CodeCommit for source code
- CodeBuild to build and test the app
- CodePipeline to orchestrate the stages
- S3 to host a static website as a deployment target

```yaml
cicd-pipeline/
│
├── main.tf
├── variables.tf
├── outputs.tf
├── buildspec.yml              # CodeBuild build instructions
├── index.html                 # Static site demo
└── app/                       # Optional source folder
```

### Step 1: Define Variables (variables.tf)
```yaml
variable "region" {
  default = "us-east-1"
}

variable "project_name" {
  default = "MyCICDPipeline"
}

variable "s3_bucket_name" {
  description = "S3 bucket for website hosting"
  type        = string
}
```

### Step 2: S3 Static Website (main.tf)
```yaml

provider "aws" {
  region = var.region
}

resource "aws_s3_bucket" "website_bucket" {
  bucket = var.s3_bucket_name

  website {
    index_document = "index.html"
  }

  acl = "public-read"
}

resource "aws_s3_bucket_policy" "bucket_policy" {
  bucket = aws_s3_bucket.website_bucket.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = "*",
      Action    = "s3:GetObject",
      Resource  = "${aws_s3_bucket.website_bucket.arn}/*"
    }]
  })
}
```

### Step 3: CodeCommit Repo
```yaml
resource "aws_codecommit_repository" "repo" {
  repository_name = "${var.project_name}-repo"
  description     = "Code repository for CI/CD pipeline"
}
```


### Step 4: IAM Roles for CodeBuild & CodePipeline
```yaml
resource "aws_iam_role" "codebuild_role" {
  name = "${var.project_name}-codebuild-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "codebuild.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "codebuild_policy" {
  role       = aws_iam_role.codebuild_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_iam_role" "codepipeline_role" {
  name = "${var.project_name}-pipeline-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "codepipeline.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "codepipeline_policy" {
  role       = aws_iam_role.codepipeline_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSCodePipelineFullAccess"
}
```

### Step 5: CodeBuild Project
```yaml
resource "aws_codebuild_project" "build" {
  name          = "${var.project_name}-build"
  service_role  = aws_iam_role.codebuild_role.arn
  build_timeout = 5

  source {
    type      = "CODECOMMIT"
    location  = aws_codecommit_repository.repo.clone_url_http
    buildspec = "buildspec.yml"
  }

  artifacts {
    type = "NO_ARTIFACTS"
  }

  environment {
    compute_type                = "BUILD_GENERAL1_SMALL"
    image                       = "aws/codebuild/standard:5.0"
    type                        = "LINUX_CONTAINER"
    privileged_mode             = true
  }
}
```

### Step 6: CodePipeline Setup
```yaml
resource "aws_codepipeline" "pipeline" {
  name     = "${var.project_name}-pipeline"
  role_arn = aws_iam_role.codepipeline_role.arn

  artifact_store {
    location = aws_s3_bucket.website_bucket.bucket
    type     = "S3"
  }

  stage {
    name = "Source"

    action {
      name             = "Source"
      category         = "Source"
      owner            = "AWS"
      provider         = "CodeCommit"
      version          = "1"
      output_artifacts = ["source_output"]

      configuration = {
        RepositoryName = aws_codecommit_repository.repo.repository_name
        BranchName     = "main"
      }
    }
  }

  stage {
    name = "Build"

    action {
      name             = "Build"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      input_artifacts  = ["source_output"]
      output_artifacts = ["build_output"]
      version          = "1"

      configuration = {
        ProjectName = aws_codebuild_project.build.name
      }
    }
  }

  stage {
    name = "Deploy"

    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "S3"
      input_artifacts = ["build_output"]
      version         = "1"

      configuration = {
        BucketName = aws_s3_bucket.website_bucket.bucket
        Extract    = "true"
      }
    }
  }
}
```

### Step 7: Create buildspec.yml
```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 14
  build:
    commands:
      - echo "Building project"
      - mkdir output
      - cp index.html output/

artifacts:
  files:
    - '**/*'
  base-directory: output
```

### Step 8: Output Website URL (outputs.tf)
```yaml
output "website_url" {
  value = aws_s3_bucket.website_bucket.website_endpoint
}
```

### Sample HTML (index.html)
```html

<!DOCTYPE html>
<html>
  <head><title>Hello from CI/CD</title></head>
  <body><h1>Deployed via AWS CodePipeline!</h1></body>
</html>
```


### Execution Steps
```bash
terraform init
terraform apply -auto-approve -var="s3_bucket_name=my-cicd-site-unique123"
```

### After apply:
- Visit the URL shown in the website_url output.
- Clone the CodeCommit repo.
- Push updated index.html to trigger the pipeline.