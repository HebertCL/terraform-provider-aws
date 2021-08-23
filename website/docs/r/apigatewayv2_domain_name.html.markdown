---
subcategory: "API Gateway v2 (WebSocket and HTTP APIs)"
layout: "aws"
page_title: "AWS: aws_apigatewayv2_domain_name"
description: |-
  Manages an Amazon API Gateway Version 2 domain name.
---

# Resource: aws_apigatewayv2_domain_name

Manages an Amazon API Gateway Version 2 domain name.
More information can be found in the [Amazon API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html).

-> **Note:** This resource establishes ownership of and the TLS settings for
a particular domain name. An API stage can be associated with the domain name using the `aws_apigatewayv2_api_mapping` resource.

## Example Usage

### Basic

```terraform
resource "aws_apigatewayv2_domain_name" "example" {
  domain_name = "ws-api.example.com"

  domain_name_configuration {
    certificate_arn = aws_acm_certificate.example.arn
    endpoint_type   = "REGIONAL"
    security_policy = "TLS_1_2"
  }
}
```

### Associated Route 53 Resource Record

```terraform
resource "aws_apigatewayv2_domain_name" "example" {
  domain_name = "http-api.example.com"

  domain_name_configuration {
    certificate_arn = aws_acm_certificate.example.arn
    endpoint_type   = "REGIONAL"
    security_policy = "TLS_1_2"
  }
}

resource "aws_route53_record" "example" {
  name    = aws_apigatewayv2_domain_name.example.domain_name
  type    = "A"
  zone_id = aws_route53_zone.example.zone_id

  alias {
    name                   = aws_apigatewayv2_domain_name.example.domain_name_configuration[0].target_domain_name
    zone_id                = aws_apigatewayv2_domain_name.example.domain_name_configuration[0].hosted_zone_id
    evaluate_target_health = false
  }
}
```

### Private CA Domain Name Configuration

```terraform
data "aws_acmpca_certificate_authority" "example" {
  arn = "arn:aws:acm-pca:us-east-1:123456789012:certificate-authority/12345678-1234-1234-1234-123456789012"
}

resource "aws_acm_certificate" "private" {
  domain_name               = "example.com"
  certificate_authority_arn = data.aws_acmpca_certificate_authority.example.id
}

resource "aws_acm_certificate" "cert" {
  domain_name       = "example.com"
  validation_method = "DNS"
}

data "aws_route53_zone" "example" {
  name         = "example.com"
  private_zone = false
}

resource "aws_route53_record" "example" {
  for_each = {
    for dvo in aws_acm_certificate.cert.domain_validation_options : dvo.domain_name => {
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
  zone_id         = data.aws_route53_zone.example.zone_id
}

resource "aws_acm_certificate_validation" "example" {
  certificate_arn         = aws_acm_certificate.cert.arn
  validation_record_fqdns = [for record in aws_route53_record.example : record.fqdn]
}

resource "aws_apigatewayv2_domain_name" "example" {
  domain_name = "http-api.example.com"

  domain_name_configuration {
    certificate_arn                         = aws_acm_certificate.private.arn
    ownership_verification_certificate_arn  = aws_acm_certificate.cert.arn
    endpoint_type                           = "REGIONAL"
    security_policy                         = "TLS_1_2"
  }
}
```

## Argument Reference

The following arguments are supported:

* `domain_name` - (Required) The domain name. Must be between 1 and 512 characters in length.
* `domain_name_configuration` - (Required) The domain name configuration.
* `mutual_tls_authentication` - (Optional) The mutual TLS authentication configuration for the domain name.
* `tags` - (Optional) A map of tags to assign to the domain name. If configured with a provider [`default_tags` configuration block](/docs/providers/aws/index.html#default_tags-configuration-block) present, tags with matching keys will overwrite those defined at the provider-level.

The `domain_name_configuration` object supports the following:

* `certificate_arn` - (Required) The ARN of an AWS-managed certificate that will be used by the endpoint for the domain name. AWS Certificate Manager is the only supported source.
Use the [`aws_acm_certificate`](/docs/providers/aws/r/acm_certificate.html) resource to configure an ACM certificate.
* `endpoint_type` - (Required) The endpoint type. Valid values: `REGIONAL`.
* `security_policy` - (Required) The Transport Layer Security (TLS) version of the [security policy](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-custom-domain-tls-version.html) for the domain name. Valid values: `TLS_1_2`.
* `ownership_verification_certificate_arn` - (Optional) The ARN of the public certificate issued by ACM to validate ownership of your custom domain. Only required when configuring mutual TLS and using an ACM imported or private CA certificate ARN as the RegionalCertificateArn.
* `hosted_zone_id` - (Computed) The Amazon Route 53 Hosted Zone ID of the endpoint.
* `target_domain_name` - (Computed) The target domain name.

The `mutual_tls_authentication` object supports the following:

* `truststore_uri` - (Required) An Amazon S3 URL that specifies the truststore for mutual TLS authentication, for example, `s3://bucket-name/key-name`.
The truststore can contain certificates from public or private certificate authorities. To update the truststore, upload a new version to S3, and then update your custom domain name to use the new version.
* `truststore_version` - (Optional) The version of the S3 object that contains the truststore. To specify a version, you must have versioning enabled for the S3 bucket.

## Attributes Reference

In addition to all arguments above, the following attributes are exported:

* `id` - The domain name identifier.
* `api_mapping_selection_expression` - The [API mapping selection expression](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-selection-expressions.html#apigateway-websocket-api-mapping-selection-expressions) for the domain name.
* `arn` - The ARN of the domain name.
* `tags_all` - A map of tags assigned to the resource, including those inherited from the provider [`default_tags` configuration block](/docs/providers/aws/index.html#default_tags-configuration-block).

## Timeouts

`aws_apigatewayv2_domain_name` provides the following [Timeouts](https://www.terraform.io/docs/configuration/blocks/resources/syntax.html#operation-timeouts) configuration options:

- `create` - (Default `10 minutes`) Used for creating the domain name
- `update` - (Default `60 minutes`) Used for updating the domain name

## Import

`aws_apigatewayv2_domain_name` can be imported by using the domain name, e.g.

```
$ terraform import aws_apigatewayv2_domain_name.example ws-api.example.com
```
