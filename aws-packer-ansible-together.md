
## Introduction

If you expect a complete Ansible or Packer tutorial, there will not be. It's just a loose set of thoughts. I will show you how to set up your environment to use Packer and Ansible in the AWS cloud.

## What is packer? Why to use it?

Packer is a software, which helps you build prebuilt system images. A typical use-case for that software is, you want to create some template for a system you are going to start in the future.

## Ansible

Ansible is very similar software to Packer. Ansible provides you way to describe your server as a code. Description is declarative. Thats mean you describe what you want (not how) and ansible provides optimal way to do it.

## Packer vs Ansible?

The Packer is classified as an Infrastructure build tool, but Ansible is Configuration automation tools. Hmmm, but what's the difference? The most important practical difference is that Ansible can be used to update the running server. The Packer can be applied to creating an operating system image. The template is created before the system is started and cannot be changed in the future.


```
https://www.packer.io/intro/getting-started/install.html

resource "aws_iam_policy" "packer_permissions_policy" {
  name        = "test-policy"
  description = "A test policy"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "ec2:Describe*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
EOF
}
```
