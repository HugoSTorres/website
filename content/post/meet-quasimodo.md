+++
Description = "Introducing quasimodo"
Tags = []
date = "2016-08-11T08:43:08-04:00"
title = "meet quasimodo"

+++

Quasimodo is a little tool I wrote for easy publishing of Hugo sites to S3.

As joshuaboelter mentioned on [reddit][1], the AWS CLI has the sync command that
allows for easy uploads to S3. The sync command is great. I've used it before.

In terms of effort, I don’t think it was significantly harder than it would
have been to just script the CLI. I wrote the first version pretty quickly.
The CLI version of Quasimodo isn’t exactly a one liner
either, because he checks to see if the bucket is properly configured to
host a static site.

Also, when I wrote Quasimodo, I had in the back of my mind the scenario
where I switch from S3 to something else. At that point, I could just
give him subcommands for every provider and keep using the same program.
To me, Quasimodo could publish Hugo sites to anywhere.

Anyways, that’s Quasimodo. Check him out on [github][2].

[1]: https://www.reddit.com/r/golang/comments/4x0jd6/github_juicemiaquasimodo_easily_deploy_hugo_sites/
[2]: https://github.com/juicemia/quasimodo