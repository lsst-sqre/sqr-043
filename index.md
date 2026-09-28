# rubin.community forum operations guide

**This technote is not yet published.**

This is an operations guide for the Rubin Observatory's Discourse forum, hosted at https://rubin.community. This guide contains playbooks for common administrative operations as well as an overview of how the Discourse forum and associated services are configured.

# rubin.community forum

## Playbooks

TBD.
This author (Adam Thornton) has no experience as a forum content administrator.
Someone with experience should write this section.

## Hosting

The Discourse forum is hosted by [Discourse](https://www.discourse.org).
As of late September 2026, we are using their Pro hosting plan.

## URLs

The Discourse forum's actual address is https://www.rubin.community.
The apex domain, https://rubin.community is pointed to it, as is the legacy URL that served the self-hosted forum, https://community.lsst.org.
This last URL is explicitly not guaranteed to remain active indefinitely.
At some point in the future, when we have determined that a sufficient proportion of traffic is referencing [rubin.community](https://rubin.community) rather than community.lsst.org, we will turn off that redirection.
We have not yet established the criteria defining "sufficient proportion".

### Redirection

We are using Google's [Firebase Hosting](https://firebase.google.com/docs/hosting) to provide the redirection services that allow accessing the forum at the domain root and via links to its former URL.

This decision was made purely from expediency: it was easy to set up and at our current usage levels we are unlikely to ever incur charges.
We are on the "Blaze" pay-as-you-go plan.
That gives us 10GiB per month of traffic, which doesn't seem like a lot; however, that's only for traffic to the redirected domains, so it's only the first connection to either the apex or community.lsst.org, and the content is simply a 301 redirect, a great many of which will fit into 10GiB.
Thereafter, the user's browser will follow the redirect to [www.rubin.community](https://www.rubin.community), and therefore will not touch the Google Firebase setup at all.

Of course, Google does not have the best track record of long-term support for its products.
However, effectively any setup that manages TLS certificates for the domains being redirected and issues an HTTP 301 redirect to the actual target site will work.
At our current level of traffic we could even do this ourselves with a simple static web server hosted on [Roundtable](https://roundtable.lsst.cloud).
However, as long as there is an inexpensive solution that leverages someone else's CDN, we might as well use it.

### Firebase Hosting setup

We added Firebase to our existing [Roundtable](https://roundtable.lsst.cloud) project as described in https://firebase.google.com/docs/web/setup#create-firebase-project under the `Existing Cloud project` tab.

Firebase is not currently Terraformed; Terraform support for Firebase is nascent at best.
At the time of writing, there did not seem to be a way to enable forwarding support via Terraform, which is the only part of Firebase we actually use.

In order to set up redirection, we simply created a new Firebase application (in our case, `rubin-apex-redirect-prod`) from the [Firebase Console](https://console.firebase.google.com/project/roundtable-prod-f6fd/) and then used the `Add custom domain` button twice; once for each redirection (one from [rubin.community](https://rubin.community) to [www.rubin.community](https://www.rubin.community), and one from [community.lsst.org](https://community.lsst.org) to [www.rubin.community](https://www.rubin.community)).

That process displayed DNS modifications we would need on the console.
Since both [lsst.org](https://lsst.org) and [rubin.community](https://rubin.community) are administered at Route 53, the process was straightforward.

The only subtlety was that [lsst.org](https://lsst.org) is not controlled by the SQuaRE team but by Rubin IT, so we had to coordinate with them to make the DNS updates.

Once the DNS updates were done, it took approximately 30 minutes for new TLS certificates to be minted, at which point, the redirection was in place.

## Content migration

Content migration was straightforward.

We put the old self-hosted Discourse server at Digital Ocean (that is, the legacy community.lsst.org) into read-only mode and took a final backup.

We downloaded that backup locally, uploaded it to the hosted server (via the Admin&#8594;Backups menu), and then requested that the Discourse admins restore the contents.

After that, the only real reconfiguration needed was to update the GitHub OAuth client ID and secret to reflect the new endpoints.

### Slightly more truthful account

In reality, it was not quite that straightforward.
Somehow, during the course of hosting our own Discourse forums, we had managed to end up with a few "unique" keys that weren't.
We discovered this by attempting to manually restore those backups to a different self-hosted image, which failed.
After diagnosing the issue, we were able to do minor database surgery to deduplicate the keys, and once that surgery was done on the running forum database (rather than the backup dumps), subsequent backups could be cleanly restored.

Discourse has done an excellent job with schema migration: a modern Discourse instance has no difficulty importing those backups and making them work in a modern version of the software.

Alas, the same was not true of the forum software itself.
The old self-hosted forum server was no longer able to be updated to a current version of the Discourse software.

### Tech debt in the legacy self-hosted server

There was no packaged version of Ruby that supported the ancient OS we were running, so trying to upgrade the forum software would eventually fail because minimum version requirements of various gems could not be met.
Experiments on a cloned server revealed that there was no way to get current from the place we were starting: one of the OS updates along the way simply failed and left the system in an unbootable state.

It was not even clear that the forum would survive a reboot.
Likely it would have, since the cloned machine would indeed boot until we tried to update it, but nevertheless it was fortunate we went ahead with the migration when we did, before disaster struck.

### Remaining task

After a burn-in period (still ongoing in late September 2026) we will shut down the self-hosted server (which is no longer addressible as community.lsst.org) and release the resources it is consuming.

## Discourse settings

### Staff Seats

Our plan allows us 15 staff seats.

This number includes both site administrators and global moderators.

#### Category moderators

However, it does not include category moderators.

Therefore, what we did was to create a `CST` group for members of the Community Science Team, and then assign the `CST` group to have moderation powers for all topics within the `Science` and `Support` categories of the forum (and their subcategories).

This permits the Community Science Team to moderate all topics of interest to them while keeping our assigned staff seats comfortably under the limit of 15.

### Local modifications

The modifications to site settings are those you would expect: the forum title, description, and logos are of course customized, as is the top-level contact email; effectively, all Rubin-specific branding and styling.

The GitHub client ID and client secret, which allow using GitHub as an OAuth authentication source, are necessarily custom as well.
So are the Slack integration tokens allowing the forum to post messages to Slack.

Besides these changes, we require moderator approval for users in trust level 0.
This, sadly, proved to be necessary, due to drive-by spamming.

### Major changes from self-hosted forum

There are several changes relative to our previous state visible to administrators.
These are generally simplifications that lessen our support burden.

* We no longer need any of the DMARC/DKIM mail delivery records we used to coordinate mail delivery via MailChimp -- Discourse now directly handles our outbound mail. These records have been dropped from DNS.
* We are applying new changes as they become available via the `Latest` setting in Admin&#8594;Manage updates. After the hideous mess of technical debt we accumulated with a self-hosted server, it is much better to simply let Discourse apply changes as they are available (also, we would need an Enterprise subscription to choose a different update model).
* No explicit backups. Discourse takes a backup every twelve hours for disaster recovery. However, we do now need a Discourse admin to restore a prior backup, because we no longer have direct write access to our forum's database (the Data Explorer plugin provides a read-only general query function).

## Emergency admin login

If something should happen to the forum such that we lose the ability to use GitHub OAuth, hosted Discourse provides a [mechanism for admin logons](https://www.rubin.community/u/admin-login).

We do not anticipate ever needing this again: it was necessary when restoring the content from a self-hosted server to a Discourse-hosted server, because the GitHub OAuth callback endpoint no longer matched the server URL.

As long as Discourse mail gets correctly delivered to noirlab.edu, the email generated with the Admin Login link will arrive, and that email will contain a magic link to log in as an admin.

It is also possible to configure local passwords for users, and the author has done this for himself, for a less fingernail-gnawing admin login experience in the absence of GitHub OAuth (historically, mail delivery to [lsst.org](https://lsst.org)/[noirlab.edu](https://noirlab.edu) has been less than entirely reliable, although now that mail is hosted by Google, that does seem to have improved).
