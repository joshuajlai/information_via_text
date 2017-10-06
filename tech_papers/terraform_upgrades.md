# Terraform Upgrades
Simple page to track what I had to do for upgrades. This is so I can look up what I did later without
being tied to a repo.

## 0.8.8 -> 0.10.5
This was painful due to the architecture changes and functionality changes
### Remote State Change
#### New Backend Block
Remote State used to be managed by `terraform remote state push` or something to that akin. Before every
terraform operation, you needed to do a `state pull` to make sure your local state was updated, run
your command (`plan` or `apply`), and then `state push`. This has changed to be fully internal to
terraform as long as a `backend` resource is defined. Ex:
```
terraform {
  backend "s3" {
    region="us-west-1"
    bucket="josh_bucket"
    key="resource_name.tfstate"
  }
}
```

Because of the change in remote state management, you need this block with your other `tf` files.
Just copy a vanilla file called `backend.tf` into every terraform project that you have, if you have
a ton of projects.

#### Init Changes
With the new backend block, you also need to `init` your projects. This will take the existing local
state file and send it to the configured backend. It also wipes out the local state file of information
so make sure you have a backup and access to the remote state. I found out about this after the fact
so I was thankful for having backups and a new destination from the old bucket.

### Plugin Changes
#### Plugin Installation/Management
Everything's a Plugin! YAY! Not really. The individual providers are now plugins so if you're using `aws`,
then terraform will install the `aws` provider plugin. Terraform defaults plugin installation at the
project level so if you have 50 projects, that's 50 installations of the plugin. The aws plugin is
around 100MB so...yeah. That sucks.

An alternative is to specify the plugin location during your `init` call. This is cool because terraform
will load a `json` file into the `.terraform` dir that points to the alternative plugin location.
terraform DOESN'T have a mechanism for install TO a directory so you'll need to manually install it
to that directory. It's a strange "missing feature" since all of the necessary GO code is there.

Btw, if you use the alternative approach, you can't use `terraform validate`. `validate` doesn't load
the `json` file and find the correct location of the plugins. This is a known bug:
https://github.com/hashicorp/terraform/issues/15916

My solution to the problem was pretty hacky. Install the plugins to a base directory of the repo and
create a symlink within the `.terraform` dir. Pretty hacky but whatevers.

#### Plugin Versions
Since providers are not plugins, terraform suggests (aka you should really do this) to pin the
version of your plugins. You do this with:
```
provider "terraform" {
  version = "~> 1.0"
}
```

Solution: create some default file you can copy around to new projects with that block or have
some dynamic system to write out a `.tf` file prior to every terraform operation (if you do this,
please open source it cuz that'd be annoying to write/manage).
Something to note is the `aws` provider also needs a `version` property. You will need to go to
every invocation of it and drop the version in. Pretty annoying but there are worse things.

### Validate Changes
The `validate` changes were pretty helpful to find all of the other changes. One major change to
validate is the fact that it tries to "plan" your terraform code. I didn't see any major
behavior changes between the new `validate` and `plan`, they seem to do almost the same thing.
Variable interpolations and provider/plugin loading all seem to be executed. There's a way to
disable variable interpolations with a flag but then it doesn't validate a whole lot.

Because of this change, validating modules (stand alone, non-complete terraform code) can't
get validated without the flag. I don't know how useful that actually is anymore. I forgot
to re-implement that.

Other changes are `validate` will try to load the project's state, including remote state. You
cannot run `validate` without the `init`, and `init` configures state. Keep that in mind if your
`validate` keeps failing and it's because you can't reach remote state.

## Conclusion
I wish I found a blog post about this. It would've saved me a lot of headache and self discovery.
But also, I should've read the docs/upgrade guide better. It contains a lot of nuanced information
that would've prevented my own headache so...ultimately I blame myself. Screw you Joshua from
yesterday. Learn to read, jerk!
