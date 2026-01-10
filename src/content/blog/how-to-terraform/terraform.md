---
title: "The secret Terraformula"
summary: "My advice on using and structuring Terraform for larger projects."
date: "Jan 7 2026"
tags:
- Programming
- Guide
- Terraform
- AWS
---

There are a thousand articles and tutorials on what Terraform is and why you should use it. 

In my experience, these are great for small hobby projects and getting started with infrastructure as code (IAC), but they often include little in the way of how to structure and work on your codebase for a larger project.

So, I want to fill that gap with this; my learnings from working with terraform in industry and for personal projects.

It will be written as the guide I wish I had found when first starting a large Terraform project, but I want this post to be a living document. I would love to know where I am going wrong so let me know by saying hello@joshuahitchon.com.  

I will assume you know what Terraform is and how to write and deploy simple configurations. If you don't, I reccommend the following, which you should check out then pop back and continue reading:

- [Terraform in 100 seconds](https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://www.youtube.com/watch%3Fv%3DtomUWcQ0P3k&ved=2ahUKEwj9jbfRwYGSAxWNbEEAHXE5Mh4QtwJ6BAgQEAI&usg=AOvVaw1mapT5rjQ04fLraxPKT_4e)
- [Terraform docs](https://developer.hashicorp.com/terraform)
- [Terraform best practices](https://medium.com/@bijit211987/mastering-terraform-best-practices-to-elevate-your-workflow-8804e801a75a)


---

## Using Terraform

Use [workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces). They are a great way to group and conceptualise infrastructure. I suggest workspaces like 'dev', 'test', and 'prod', or even 'jira-123'.

You can create one like so:
```sh
terraform workspace new <workspace-name>
```

By default, each workspace gets its own [state file](https://developer.hashicorp.com/terraform/language/state). This means you have separation between those environments mentioned and so they shouldn't affect one another.

You should configure the state file be stored somewhere shared like S3, **not** locally. That way you can work on infrastructure with others, and do not risk losing the state file when you leave your laptop on the train. 

You should version this file (S3 offers this) so you can easily rollback if you mess up the state.

> Note if you use S3, you can be really clever by having terraform create its own bucket to store its state file. Don't - keep it simple, and just make one manually.

If you do not trust yourself or your fellow devs (which you shouldn't), you may consider setting up a separate AWS account to deploy the 'prod' workspace. This gives:
1. an extra layer of isolation from dev infrastructure
2. the ability to more strictly guard access to managing and creating AWS resources in prod. (Ideally you can limit it to just a single set of credentials, used by some build server).

Also, hopefully it goes without saying but yes setup MFA requirements for dev accounts, and *do not* use the root account, lest you welcome a problem in with open arms. 

## Repo structure
Here is a suggested repository setup:

```
src
├── services
│   ├── shop
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── providers.tf
│   │   └── backend.tf
│   └── app
|       └── ...
├── modules
│   ├── s3-bucket
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── user-pool
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── variables
    ├── dev.tfvars
    ├── prod.tfvars
    └── test.tfvars  
        
```
### Glossary for repo
| File Type        | Description                                                                 |
|------------------|-----------------------------------------------------------------------------|
| backend.tf       | Sets up how the Terraform state file will be stored.                        |
| providers.tf     | Declares required providers and dependencies (e.g., AWS).                  |
| variables.tf     | Defines input variables required by the configuration.                     |
| outputs.tf       | Exports values from the configuration, such as created resource IDs/names. |
| [*.tfvars](https://spacelift.io/blog/terraform-tfvars)         | Provides variable values for variables declared in `variables.tf`.         |
| main.tf          | Declares and configures resources; may be split into multiple files for large configurations. |

FYI, these file names are not mandatory, and ultimately they are all just `.tf` files, but I have found it is helpful to be consistent.
---

In this setup, each service is a top-level 'thing' which, for the most part, is a collection of modules.

These are declared in the `modules` directory. This separation encourages you to breakdown your configurations in a way that is easy to re-use and share in other projects. If you have multiple cloud projects, you could even create a separate repo from just this directory, and have only a single service at the root.

However, in my opinion the more I have separated my codebase like this, the more headaches it has caused, just to keep things mildly cleaner. It is much simpler to keep everything together unless you have a specific reason.

You may wish to include a providers file in your module if they have further specific dependencies, but I have often not found this needed.

I have found the `variables` directory a very helpful structure. This setup creates a natural way of declaring variables for your type of deployment. I have often found that this pairs up with your workspace.

You can then easily tweak the type of deployment you are doing.
```sh
terraform workspace select <workspace-name>
terraform apply -var-file=../../../variables/<workspace-name>.tfvars
```

## Handling multi-region deployments:

Often online, I have seen it advised to pass the region as a variable, so you can reuse the same terraform code across regions. This is a solid approach, especially if there will be no differences between regions. 

However, on a larger project, you may well find your requirements could evolve where you need different resources and services in each region. 

I would suggest break the sacred DRY principle in this instance; by duplicating the top-level service directory e.g. into shop-eu-west-1, and shop-us-east-1. You can then pull any common infrastructure into a module, and declare other resources as needed. 

This also has the advantage of making things feel a bit safer, in my opinion, where it is a lot harder to for example deploy to the wrong region if you forget to set the right variable.

## Tests

> Yes, you should be writing tests for large projects, and terraform is no different.

There are a few test frameworks out there, including terraform itself which includes some testing capabilities. Personally, when using this I found it odd to write tests in the same declaritive style as my configurations, as I often seemed to just be writing out a very similar file to the thing I was testing. But it is already included in terraform, and it's the same language, so maybe give it a go and see what you think.

I reccomend Terratest. It is written in Go, and has a lot of fancy features, chief among them being you can run tests that deploy and teardown *real* infrastructure and actually try using it with your configuration.

I have found Terratest particularly helpful for integration tests. I would warn they are quite slow and can have costs (as it is real infrastructure). Therefore, I usually try to keep it to happy path or particularly paranoid potential error cases.

I could write a lot about Terratest, and may write a future guide on this because there is a similar level of good resources for getting started, but not much beyond that.

## Lambdas (Serverless functions)
*This is more an AWS specific section.*

If you are using AWS lambdas or an equivelant, I reccommend keeping your lambdas in a separate top level directory of your repository: 

```
src
├── services
├── modules
├── lambdas
│    ├── src
│    └── test
└── variables
```

I have found this approach feels tidier than having lambda code scattered across modules, and makes it much easier to run tools like unit tests and coverage scanners.

> Note, yes please write unit tests for your lambdas, even if they are small. They save so much heartache.

You will likely find you will end up writing very similar code between lambdas. If using AWS, I reccomend writing a helper package and putting it into a lambda layer.

On languages, it doesn't really matter. Ideally use something with lots of support (like Python and NodeJS) which have an AWS supported runtime. I would warn of using more obscure languages, as AWS has a habit of depreciating runtimes (like Go recently https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html#runtimes-deprecated).

## Final thoughts

That is pretty much it for now. I'll let you know if add anything new below:
- 10/01/2026 - Not yet

As some final quick fire advice:
- Write scripts to automate things like deployment. You will be faster and less likely to make a mistake.
- Use a CI pipeline to test and deploy infrastructure (using your scripts). Just like any other project, this will reduce the chances of regressions and mistakes.

I hope you found this guide helpful! Let me know your thoughts by saying hello@joshuahitchon.com.

\- Josh