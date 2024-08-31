# Creating SSL Certificates using AWS Certificate Manager with DNS Validation using Terraform

Securing online communication is vital for protecting sensitive data. One effective way to establish secure connections is by using SSL/TLS certificates, which enable HTTPS for your websites and applications. AWS Certificate Manager (ACM) simplifies the process of provisioning, managing, and deploying these certificates. ACM takes care of the complexities of certificate renewal and supports both public and private certificates.

To further enhance security, DNS validation can be used as a method to confirm domain ownership when issuing certificates. DNS validation involves adding a specific DNS record to your domain's configuration, which ACM then verifies. This approach is particularly useful for automating the validation process, as it avoids the need for manual intervention.

In this blog post, we'll explore how to leverage Terraform to automate the creation of an SSL/TLS certificate with ACM and implement DNS validation for secure and streamlined certificate management.

## Architecture Overview
Before diving into the implementation, let’s outline the architecture:

![alt text](/images/diagram.png)

## Step 1: Request an ACM Certificate
Use the aws_acm_certificate resource to request an SSL/TLS certificate. Specify the domain name for which you want the certificate.

```terraform
resource "aws_acm_certificate" "mycert_acm" {
  domain_name               = "ec2.${var.domain_name}"
  subject_alternative_names = ["*.ec2.${var.domain_name}"]

  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}
```

## Step 2: Create Route 53 DNS Record for Validation
First, ensure you have a Route 53 hosted zone for your domain. To validate the certificate via DNS, use the aws_route53_record resource. This creates the necessary DNS record that AWS will check to verify domain ownership.

```terraform
data "aws_route53_zone" "selected_zone" {
  name         = var.domain_name
  private_zone = false
}

resource "aws_route53_record" "cert_validation_record" {
  for_each = {
    for dvo in aws_acm_certificate.mycert_acm.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.selected_zone.zone_id
}
```

## Step 3: Handle Certificate Validation
Once the DNS records are in place, ACM will automatically check the DNS entries. You can use Terraform to wait for the validation to complete.

```terraform
resource "aws_acm_certificate_validation" "cert_validation" {
  timeouts {
    create = "5m"
  }
  certificate_arn         = aws_acm_certificate.mycert_acm.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation_record : record.fqdn]
}
```

## Steps to Run Terraform
Follow these steps to execute the Terraform configuration:
``` terraform
terraform init
terraform plan 
terraform apply -auto-approve
```

Upon successful completion, Terraform will provide relevant outputs.
```terraform
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

## Testing
ACM Certificate with status = issued
![alt text](/images/acm_cert.png)

![alt text](/images/acm_cert_1.png)

Route53 CNAME record for domain in consideration
![alt text](/images/route53_cname.png)

## Cleanup
Remember to stop AWS components to avoid large bills.
```terraform
terraform destroy -auto-approve
```
## Conclusion
By leveraging Terraform and AWS Certificate Manager (ACM), you can streamline the process of provisioning and managing SSL/TLS certificates with DNS validation. This approach not only automates the creation and validation of certificates but also simplifies ongoing maintenance by handling renewals through ACM.

## Resources
Terraform Documentation: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/acm_certificate_validation

GitHub Repo: https://github.com/chinmayto/terraform-aws-acm-ssl-cert-dns-validation
