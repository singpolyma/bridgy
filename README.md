![Bridgy](https://raw.github.com/snarfed/bridgy/master/static/bridgy_logo_thumb.jpg) [Bridgy](https://brid.gy/) [![Circle CI](https://circleci.com/gh/snarfed/bridgy.svg?style=svg)](https://circleci.com/gh/snarfed/bridgy)
===

Got a web site? Want social network replies and likes on your site? Want to post and tweet from your site? Bridgy is for you.

https://brid.gy/

Bridgy pulls comments and likes from social networks back to your web site. You
can also use it to publish your posts to those networks.
[See the docs](https://brid.gy/about) for more details.

License: This project is placed in the public domain.


Development
---
You'll need the
[App Engine Python SDK](https://cloud.google.com/appengine/downloads#Google_App_Engine_SDK_for_Python)
version 1.9.15 or later (for
[`vendor`](https://cloud.google.com/appengine/docs/python/tools/libraries27#vendoring)
support). Add it to your `$PYTHONPATH`, e.g.
`export PYTHONPATH=$PYTHONPATH:/usr/local/google_appengine`, and then run:

```
virtualenv local
source local/bin/activate
pip install -r requirements.freeze.txt

# We install gdata in source mode, and App Engine doesn't follow .egg-link
# files, so add a symlink to it.
ln -s ../../../src/gdata/src/gdata local/lib/python2.7/site-packages/gdata
ln -s ../../../src/gdata/src/atom local/lib/python2.7/site-packages/atom

python -m unittest discover
```

The last command runs the unit tests. If you send a pull request, please include
(or update) a test for the new functionality if possible!

If you hit an error during setup, check out the [oauth-dropins Troubleshooting/FAQ section](https://github.com/snarfed/oauth-dropins#troubleshootingfaq). For searchability, here are a handful of error messages that [have solutions there](https://github.com/snarfed/oauth-dropins#troubleshootingfaq):

```
bash: ./bin/easy_install: ...bad interpreter: No such file or directory

ImportError: cannot import name certs

ImportError: No module named dev_appserver

ImportError: cannot import name tweepy

File ".../site-packages/tweepy/auth.py", line 68, in _get_request_token
  raise TweepError(e)
TweepError: must be _socket.socket, not socket

error: option --home not recognized
```

There's a good chance you'll need to make changes to
[granary](https://github.com/snarfed/granary),
[oauth-dropins](https://github.com/snarfed/oauth-dropins), or
[webmention-tools](https://github.com/snarfed/webmention-tools) at the same time
as bridgy. To do that, clone their repos elsewhere, then install them in
"source" mode with:

```
pip uninstall oauth-dropins
pip install -e <path to oauth-dropins>
ln -s <path to oauth-dropins>/oauth_dropins \
  local/lib/python2.7/site-packages/oauth_dropins

pip uninstall granary
pip install -e <path to granary>
ln -s <path to granary>/granary \
  local/lib/python2.7/site-packages/granary

pip uninstall webmention-tools
# webmention-tools isn't in pypi
ln -s <path to webmention-tools>/webmentiontools \
  local/lib/python2.7/site-packages/webmentiontools
```

The symlinks are necessary because App Engine's `vendor` module evidently
doesn't follow `.egg-link` or `.pth` files. :/

This command runs the tests, pushes any changes in your local repo, and
deploys to App Engine:

```shell
cd ../oauth-dropins && source local/bin/activate.csh && python -m unittest discover && \
  cd ../granary && source local/bin/activate.csh && python -m unittest discover && \
  cd ../bridgy && source local/bin/activate.csh && python -m unittest discover && \
  git push && md5sum -c keys.md5 && ~/google_appengine/appcfg.py update .
```

[`remote_api_shell`](https://cloud.google.com/appengine/docs/python/tools/remoteapi#using_the_remote_api_shell)
is a useful interactive Python shell that can interact with the production app's
datastore, memcache, etc. To use it,
[create a service account and download its JSON credentials](https://console.developers.google.com/project/brid-gy/apiui/credential),
put it somewhere safe, and put its path in your `GOOGLE_APPLICATION_CREDENTIALS`
environment variable.


Monitoring
---

App Engine's [built in dashboard](https://appengine.google.com/dashboard?&app_id=s~brid-gy) and [log browser](https://console.developers.google.com/project/brid-gy/logs) are pretty good for interactive monitoring and debugging.

For alerting, we've set up [Google Cloud Monitoring](https://app.google.stackdriver.com/services/app-engine/brid-gy/) (née [Stackdriver](http://en.wikipedia.org/wiki/Stackdriver)). Background in #377. It [sends alerts](https://app.google.stackdriver.com/policy-advanced) by email and SMS when [HTTP 4xx responses average >.1qps or 5xx >.05qps](https://app.google.stackdriver.com/policy-advanced/650c6f24-17c1-41ac-afda-90a1e56e82c1), [latency averages >15s](https://app.google.stackdriver.com/policy-advanced/2c0006f3-7040-4323-b105-8d24b3266ac6), or [instance count averages >5](https://app.google.stackdriver.com/policy-advanced/5cf96390-dc53-4166-b002-4c3b6934f4c3) over the last 15m window.


Misc
---
The datastore is automatically backed up by a
[cron job](https://developers.google.com/appengine/articles/scheduled_backups)
that runs
[Datastore Admin backup](https://developers.google.com/appengine/docs/adminconsole/datastoreadmin#backup_and_restore)
and stores the results in
[Cloud Storage](https://developers.google.com/storage/docs/), in the
[brid-gy.appspot.com bucket](https://console.developers.google.com/project/apps~brid-gy/storage/brid-gy.appspot.com/).
It backs up all entities weekly, and all entities except `Response` and
`SyndicatedPost` daily, since they make up 92% of all entities by size and
they aren't as critical to keep.

We use this command to set a
[Cloud Storage lifecycle policy](https://developers.google.com/storage/docs/lifecycle)
on that bucket that deletes all files over 30 days old:

```
gsutil lifecycle set cloud_storage_lifecycle.json gs://brid-gy.appspot.com
```

So far, this has kept us within the
[5GB free quota](https://developers.google.com/appengine/docs/quotas#Default_Gcs_Bucket).
Run this command to see how much space we're currently using:

```
gsutil du -hsc gs://brid-gy.appspot.com/\*
```
