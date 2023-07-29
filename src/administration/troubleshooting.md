# Troubleshooting

Different problems that can occur on a new instance, and how to solve them.

Many Lemmy features depend on a correct reverse proxy configuration. Make sure yours is equivalent to our [nginx config](https://github.com/LemmyNet/lemmy-ansible/blob/main/templates/nginx.conf).

## General

### Logs

For frontend issues, check the [browser console](https://webmasters.stackexchange.com/a/77337) for any error messages.

For server logs, run `docker-compose logs -f lemmy` in your installation folder. You can also do `docker-compose logs -f lemmy lemmy-ui pictrs` to get logs from different services.

If that doesn't give enough info, try changing the line `RUST_LOG=error` in `docker-compose.yml` to `RUST_LOG=info` or `RUST_LOG=verbose`, then do `docker-compose restart lemmy`.

### Creating admin user doesn't work

Make sure that websocket is working correctly, by checking the browser console for errors. In nginx, the following headers are important for this:

```
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

### Rate limit error when many users access the site

Check that the headers `X-Real-IP` and `X-Forwarded-For` are sent to Lemmy by the reverse proxy. Otherwise, it will count all actions towards the rate limit of the reverse proxy's IP. In nginx it should look like this:

```
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

## Federation

### Other instances can't fetch local objects (community, post, etc)

Your reverse proxy (eg nginx) needs to forward requests with header `Accept: application/activity+json` to the backend. This is handled by the following lines:

```
set $proxpass "http://0.0.0.0:{{ lemmy_ui_port }}";
if ($http_accept = "application/activity+json") {
set $proxpass "http://0.0.0.0:{{ lemmy_port }}";
}
if ($http_accept = "application/ld+json; profile=\"https://www.w3.org/ns/activitystreams\"") {
set $proxpass "http://0.0.0.0:{{ lemmy_port }}";
}
proxy_pass $proxpass;
```

You can test that it works correctly by running the following commands, all of them should return valid JSON:

```
curl -H "Accept: application/activity+json" https://your-instance.com/u/some-local-user
curl -H "Accept: application/activity+json" https://your-instance.com/c/some-local-community
curl -H "Accept: application/activity+json" https://your-instance.com/post/123 # the id of a local post
curl -H "Accept: application/activity+json" https://your-instance.com/comment/123 # the id of a local comment
```

### Fetching remote objects works, but posting/commenting in remote communities fails

Check that [federation is allowed on both instances](federation_getting_started.md).

Also ensure that the time is accurately set on your server. Activities are signed with a timestamp, and will be discarded if it is off by more than 10 seconds.

### Other instances don't receive actions reliably

Lemmy uses a queue to send out activities. The size of this queue is specified by the config value `federation.worker_count`. Very large instances might need to increase this value. Search the logs for "Activity queue stats", if it is consistently larger than the worker_count (default: 64), the count needs to be increased.

# tips from https://matrix.to/#/#lemmy-support-general:discuss.online

**Connect to the database in the docker container:**

> [requires PostgreSQL-client]

– get container ip with:

```sudo docker exec container-name ip address```

– connect, requires password:

```psql -h 192.168.176.3 -p 5432 -U lemmy lemmy```

----
**View logs:**

```sudo docker-compose logs -f lemmy```

----
**doing DB dump:**

> [not tried yet]

docker-compose exec postgres pg_dumpall -c -U lemmy | gzip > lemmy_dump_date +%Y-%m-%d"_"%H_%M_%S.sql.gz

----
**Reset all logins:**

> [back up your secret first, just in case]

```SELECT * FROM secret;```

– generate a new secret

```UPDATE secret SET jwt_secret = gen_random_uuid();```

----
**Trim activity table:**

```delete from public.activity where id < (select max(id) from public.activity) - 100000;```

----
**Disable 2FA for a user:**

https://lemmyadmin.site/post/41 (https://aussie.zone/post/391892)

----

**Move pics to block storage:**
> [not tried yet]

https://git.asonix.dog/asonix/pict-rs/#user-content-filesystem-to-object-storage-migration

https://lemmy.eus/comment/165512

----

**show non-local subscribers:**
> [all bots for me]

```select distinct P.name, I.domain from community_follower AS CF left join person AS P on CF.person_id = P.id left join instance AS I on P.instance_id = I.id where not P.local;```
