# Terraforming with types

Goal: Generate Terraform HCL using a strongly-typed DSL

## History

I got really into Terraform at a previous job but a coworker and I disliked the stringly-typed nature of HCL. I asked, "What if, instead, we could we generate the HCL based on a similarly-organized DSL in our beloved Scala?" We never got around to writing a single line of it but the idea stuck with both of us.

## Examples

I do not have a large Terraform configuration, but a public one with which I am very familiar is [terraform-aws-s3-cloudfront-website](https://github.com/riboseinc/terraform-aws-s3-cloudfront-website). Pull it up and compare the expressiveness of its example with this DSL:

```scala

implicit val provMain = Provider.AWS("main", region = Region.US_West_2)
val provCf = Provider.AWS("cloudfront", region = Region.US_East_1)

val providers = Seq(provMain, provCf)

val fqdn = Variable[FQDN](
  "fqdn",
  default = FQDN("mysite.example.com"),
  description = "The fully-qualified domain name of the resulting S3 website.")
val domain = Variable[DomainName]("domain",
                                  default = DomainName("example.com"),
                                  description = "The domain name / .")
val allowed_ips = Variable[List[CIDR]](
  "allowed_ips",
  default = List(CIDR("999.999.999.999/32"))) _ // or maybe "1.1.1.1/32".to_cidr?

//how modules' parameters would be handled might have to be lower level
val modCfWebsite =
  Module("main",
         source = "github.com/riboseinc/terraform-aws-s3-cloudfront-website")(
    "fqdn" -> fqdn,
    "ssl_certificate_arn" -> "${aws_acm_certificate_validation.cert.certificate_arn}",
    "allowed_ips" -> allowed_ips,
    "index_document" -> "index.html",
    "error_document" -> "404.html",
    "refer_secret" -> hcl"""base64sha512("REFER-SECRET-19265125-${fqdn}-52865926")""",
    "force_destroy" -> true,
    "providers" -> Map(
      "aws.main" -> provMain,
      "aws.cloudfront" -> provCf
    ),
    // Optional WAF Web ACL ID, defaults to none.
    "web_acl_id" -> hcl"data.terraform_remote_state.site.waf-web-acl-id"
  )

val cert = Resource.AWS.ACM
  .Certificate(domainName = domain, validationMethod = DNS)(provCf)

// if this is a common pattern, create a ResourceRecord type that would reduce this to something like
//     ResourceRecord(cert.domainValidationOptions(0)).name
val certValidationRecord = Resource.AWS.Route53.Record(
  name = cert.domainValidationOptions(0).resourceRecordName,
  `type` = cert.domainValidationOptions(0).resourceRecordType,
  records = cert.domainValidationOptions(0).resourceRecordValue,
  zoneId = XXXXXX
)("cert_validation", provCf)
val certValidation = Resource.AWS.ACM.CertificateValidation(
  certificateArn = cert.arn,
  validationRecordFqdns = certValidationRecord.fqdn)("cert", provCf)

val zone = Data.AWS.Route53.Zone(name = domain, privateZone = false)("main") // provMain is consumed implicitly

// the output of an hcl-interpolated string would be ${contents}. The contents of ${} would be treated as
// Scala code so that variables in scope could be referenced and stringified
val web = Resource.AWS.Route53.Record(
  name = fqdn,
  `type` = DNS.A,
  zoneId = zone.id,
  alias = Route53.Alias(
    name = hcl"""${modCfWebsite}.cf_domain_name""", // modCfWebsite becomes "module.main" because of the name passed to the constructor
    zoneId = hcl"""${modCfWebsite}.cf_hosted_zone_id""",
    evaluateTargetHealth = false
  )
)
// TODO add the outputs, which are basically like
outputs += Output[AWS.S3.BucketId]("s3_bucket_id",
                                   hcl"""${modCfWebsite}.s3_bucket_id}""")
outputs += Output("route53_fqdn", web.fqdn) // the type of this would be Output[FQDN], implied because Resource.AWS.Route53.Record#fqdn would be of type FQDN.

```

## License

Don't use this, really. I'll license it appropriately if I actually make this work in any capacity.
