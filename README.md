# terragrunt-poc

This repository contains a POC to show feature based terragrunt/terraform development.

## Background

When operating on very large AWS estates consisting of many hundreds of AWS accounts, it may seem natural to setup terraform or terragrunt in a way that there is one statefile per account. In this way you can operate on each account in isolation.

However with different types of accounts necessary (eg network backbone accounts, audit accounts for security, and tenant accounts for all tenants of the org), this can quickly become cumbersome, with different "features" requiring terraforming different things in different accounts. This then requires a lot of orchestration and the need for terragrunt and/or atlantis.

A different option would be to organise terraform/terragrunt and the statefile along feature boundaries. So that a single feature (such as a cloudtrail setup, or a transit gateway connection) can have its own statefile with all accounts applied in the same place.

Doing this is naturally difficult, because it requires that different aws providers exist so that terraform can have the permissions to operate in each of the accounts it needs to. It is not currently possible to generate provider configuration dynamically in terraform using concepts such as `for_each`, `count` or `dynamic` blocks, because of terraform language constraints. Not only this, but any providers being dynamically created also need to be passed into terraform modules via the `provider{}` sub-block, which means that both providers and module calls need to be dynamically generated.

This POC aims to create a working model where both providers and module calls can be generated from a single list or multiple lists of accounts - so that for a single feature, terragrunt will spin up all the cross-account providers necessary to implement the feature across the whole org, run all the configuration everywhere, and store the state for that feature in a single statefile.

It seems others also consider feature based terraform as a possible nirvana. I found this article interesting, although he has not finished [link](https://medium.com/geekculture/from-terralith-to-terraservice-with-terraform-acf990e65578)

## POC

This particular POC assumes that you own an AWS Organization with multiple accounts, and the accounts are defined in `org.hcl`. It may be possible to look these up instead of statically defining them in such a file. In order to demonstrate cross-account features, this POC is designed for the following kind of AWS::Organizations structure setup:

```
                            .--------.
                            | master |  <-- AWS::Organization owner
                            '--------'
                                 |
       .--------------------------------------------------.
       |                 |                 |              |
.-------------.   .--------------.   .-----------.  .------------.
| tenant1-dev |   | tenant1-prod |   | audit-dev |  | audit-prod |
'-------------'-. '--------------'-. '-----------'  '------------'
  | tenant2-dev |   | tenant2-prod |
  '-------------'-. '--------------'-.
    | tenant3-dev |   | tenant3-prod |
    '-------------'   '--------------'

<--dev accounts--> <--prod accounts--> <---- audit accounts ----->
```

For demonstration purposes, it also assumes that you have the following roles available in each account, all assumable from a single IAM entity which you are already authenticated as before you run:

| account             | role                   |
| ------------------- | ---------------------- |
| master              | my-org-admin-role      |
| all tenant accounts | my-security-admin-role |
| all audit accounts  | my-security-admin-role |

Note that this POC implementation allows you to specify multiple roles and have multiple providers for each account (to do multi-provider work within an account).

The project is split up into "features" which can be found in the `features/` directory each with their own `README.md`

Requirements:

* A single identity (such as a federated IAM role, or an IAM user) which has permission to `sts:AssumeRole` into all the other accounts as various roles. Each account uses its own role, however a single starting point has to be able to automatically assume all the roles needed.

How it works:

* Although you cannot do foreach on a terraform provider, or foreach in a terragrunt generate, you can use generate a bunch of provider definitions in terragrunt using `formatlist()` within the contents of a `generate{}` block.
* A feature should be able to select which accounts it operates on. This is accomplished by being able to pass through a list of accounts from the feature to the terragrunt root where it is interpolated into providers.
* A read_terragrunt_config() defined in the root, refers to the include_path, allowing reading of feature config from the parent's perspective.
* A feature needs the ability to operate on different subsections of accounts in different ways. For example a VPC attachment to a central TransitGateway in a separate account, or a Cloudtrail propagation to a central audit account.
* To accomplish this, we define account-groups in the `feature/config.hcl`, calling a different module and provider(s) for each one.
* A reasonable way to select the individual accounts for each account-group was to use a regex match on the AWS account-names (as defined in `org.hcl`) - by naming convention
* Providers need more information than just the account names. Account IDs, roles, and regions all need specifying.
* Each account_group can specify a different target module, and a different set of providers to pass through, as well as different variables and different dependencies.
* The parent `accounts.hcl` takes all this information and generates a `providers.tf` file and `modules.tf` file.
* Terragrunt does all the heavy lifting. As soon as you're in the feature module itself, all the config is present, and you can call further modules easily.
* Two examples are included
  * `simple-iam-role`: create an IAM role in every account, but managed by one statefile
  * `cloudtrail`: lookup AWS accounts in the master account, create a bucket and policy in the `audit` accounts, and create cloudtrails in the `tenant` accounts that point to those buckets.
* Terragrunt applies them in order, because you can specify explicit cross-account-group dependencies

Further areas for development:

* Make one feature dependent on another feature (using `dependency{}`)
* Features that may require both aws and gcp providers at the same time?
* Solves the problem of having to run a terraform for each account, but it might make onboarding a new account require running all features.

Comments:

* A `foreach` on either terraform providers, or terragrunt generate blocks would help with the need to generate dynamic code - and make it more readable.

Does it work?:

* Yes it does work, but obviously I had to obfuscate my test accounts in `org.hcl` and provide placeholders instead. You should therefore not expect to be able to just run this out of the box to show the POC in action, but you should have accounts set up which you can define in `org.hcl` to demonstrate the functionality in your own org
