BitBalloon REST API
===================

BitBalloon is a hosting service for the programmable web. It understands your documents and provides an API to deploy sites, manage form submissions, inject javascript snippets and do intelligent updates of HTML documents. This is a REST-style API that uses JSON for serialization and OAuth 2 for authentication.

Making a request
----------------

All URLs start with `https://www.bitballoon.com/api/v1/`. **SSL only**. The path is prefixed with the API version. If we change the API in backward-incompatible ways, we'll bump the version marker and maintain stable support for the old URLs.

To make a request for all the sites you have access to, you'd append the sites index path to the base url to form something like https://www.bitballoon.com/api/v1/sites. In curl, that looks like:

```shell
curl -H 'User-Agent: MyApp (yourname@example.com)' https://www.bitballoon.com/api/v1/sites?access_token=oauth2_access_token
```

Authenticating
--------------

BitBalloon uses OAuth2 for authentication. All requests must use HTTPS. You'll need an application client key and a client secret before you can access the BitBalloon API. Please contact us at team@bitballoon.com for your credentials.

If you're making a public integration with BitBalloon for others to enjoy, you must use OAuth 2. This allows users to authorize your application to use BitBalloon on their behalf without having to copy/paste API tokens or touch sensitive login info.

The Oauth2 end user authorization endpoint is `https://www.bitballoon.com/oauth/authorize`.

Endpoints
---------

* `/sites` all sites
* `/sites/{site_id}/forms` all forms for a site
* `/sites/{site_id}/submissions` all submissions for a site
* `/sites/{site_id}/files` all files for a site
* `/sites/{site_id}/snippets` all snippets to be injected into the HTML of a site
* `/sites/{site_id}/metadata` a metadata object for a site (can be used in combination with the snippets)
* `/sites/{site_id}/deploys` all deploys for a site
* `/forms` all forms
* `/forms/{form_id}/submissions` all submissions from a specific form
* `/submissions` all form submissions
* `/users` all users you have access to
* `/users/{user_id}/sites` all sites for a specific user
* `/users/{user_id}/forms` all forms for a specific user
* `/users/{user_id}/submissions` all form submissions for a specific user

A note on the `site_id`: this can either be the actual `id` of a site, but it is interchangeable with the full domain for a site (some-site.bitballoon.com or site.example.com).

Rate Limiting
=============

To protect BitBalloon from getting flooded by automated deploys or misbehaving applications, the BitBalloon API is rate limited.

You can make up to 200 requests per minute.

You can check the returned HTTP headers of any API request to see your current rate limit status:

```
X-RateLimit-Limit: 200
X-RateLimit-Remaining: 56
X-RateLimit-Reset: 1372700873
```

If you need higher limits, please contact us.

Pagination
==========

Requests that return multiple items will be paginated to 100 items by default. You can specify further pages with the ?page parameter. You can also set a custom page size up to 100 with the ?per_page parameter.

Note that page numbering is 1-based and that ommiting the ?page parameter will return the first page.

### Link Header

The pagination info is included in the `Link` header.

```
Link: <https://www.bitballoon.com/api/v1/sites?page=3&per_page=20>; rel="next",
  <https://www.bitballoon.com/api/v1/sites?page=5&per_page=20>; rel="last"
```

Linebreak is included for readability.

The possible rel values are:

* `next`
  Shows the URL of the immediate next page of results.
* `last`
  Shows the URL of the last page of results.
* `prev`
  Shows the URL of the immediate previous page of results.


Sites
=====

The `/sites` endpoint allows you to access the sites deployed on BitBalloon.

Get Sites
---------

* `GET /sites` will return all sites you have access to.

```json
[
  {
    "id":"3970e0fe-8564-4903-9a55-c5f8de49fb8b",
    "premium":false,
    "claimed":true,
    "name":"synergy",
    "custom_domain":"www.example.com",
    "url":"http://www.example.com",
    "admin_url":"https://www.bitballoon.com/sites/synergy",
    "screenshot_url":null,
    "created_at":"2013-09-17T05:13:08Z",
    "updated_at":"2013-09-17T05:13:19Z",
    "user_id":{"51f60d2d5803545326000005"},
  }
]
```

Get Site
--------

* `GET /sites/3970e0fe-8564-4903-9a55-c5f8de49fb8b` will return the site from its ID
* `GET /sites/www.example.com` will return the site matching the domain `www.example.com`

```json
{
  "id":"3970e0fe-8564-4903-9a55-c5f8de49fb8b",
  "premium":false,
  "claimed":true,
  "name":"synergy",
  "custom_domain":"www.example.com",
  "notification_email:"me@example.com",
  "url":"http://www.example.com",
  "admin_url":"https://www.bitballoon.com/sites/synergy",
  "screenshot_url":null,
  "created_at":"2013-09-17T05:13:08Z",
  "updated_at":"2013-09-17T05:13:19Z",
  "user_id":{"51f60d2d5803545326000005"},
}
```

Create Site
-----------

Creating a site will initiate a new deploy. You can either create a new site with a list of files you intend to upload, or by posting a ZIP file.

### POST a ZIP file

* `POST /sites` with `zip=zip-file`. You must use `Content-Type: multipart/form-data` to send the zip file in a property called `zip`.

```json
{
  "id": "3970e0fe-8564-4903-9a55-c5f8de49fb8b",
  "subdomain": "some-autogenerated-subdomain",
  "url": "http://some-autogenerated-subdomain.bitballoon.com",
  "state": "processing",
  "required": []
}
```

This will return `201 Created` with the API URL for the new site in the `Location` header, along with a simplified JSON representation of the site. The site will be in the `processing` state and you can poll the URL in the `Location` header until the state has changed to either `ready` or `error`.

Here's an example of creating a new site from a zip file called `landing.zip` via Curl (assuming you've gone through the process to get an oauth access_token) 

```bash
curl -F "zip=@landing.zip;type=application/zip" https://www.bitballoon.com/api/v1/sites?access_token={access_token}
```

### POST a list of files

* `POST /sites` with `{files: {"/index.html": "SHA1_OF_YOUR_INDEX_HTML"}}` will create a new site in the `uploading` state. You must use `Content-Type: application/json`

The files object should contain all the files you wish to upload for this deploy, together with a SHA1 of the content of each file.

```json
{
  "id": "3970e0fe-8564-4903-9a55-c5f8de49fb8b",
  "subdomain": "some-autogenerated-subdomain",
  "url": "http://some-autogenerated-subdomain.bitballoon.com",
  "state": "uploading",
  "required": ["SHA1_OF_YOUR_INDEX_HTML"]
}
```

This will return `201 Created` with the API URL for the new site in the `Location` header, along with a simplified JSON representation of the site. The `required` property will give you a list of files you need to upload. BitBalloon will inspect the SHA1s you sent in the request. You'll only need to upload the files BitBalloon doesn't have on its servers. The `state` can be either `uploading` or `processing` depending on whether or not you need to upload any more files.

To upload any required files, use the `PUT /sites/{site_id}/files/{path}` endpoint for each file. Once all the files are uploaded, the processing of the site will begin.

Update Site
-----------

* `PATCH /sites/{site_id}` will let you update some attributes on a site
* `PUT /sites/{site_id}` will let you update some attributes on a site

This lets you update the `name`, `custom_domain`, `password` and `notification_email` fields of a site.


Destroy Site
------------

* `DELETE /sites/{site_id}` will permanently delete a site 
* `DELETE /sites/{site_domain}` will permanently delete a site

This will return `200 OK`.


Submissions
===========

The `/submissions` endpoint gives access to the form submissions of your BitBalloon sites. You can scope submissions to a specific site (`/sites/{site_id}/submissions`) or to a specific form (`/forms/{form_id}/submissions`).

Get Submissions
---------------

* `GET /submissions` will return a list of form submissions

```json
[
  {
    "id":"5231110b5803540aeb000019",
    "number":13,
    "title":null,
    "email":"test@example.com",
    "name":"Mathias Biilmann",
    "first_name":"Mathias",
    "last_name":"Biilmann",
    "company":"BitBalloon",
    "summary":"Hello, World",
    "body":"Hello, World",
    "data": {
      "email":"test@example.com",
      "name": "Mathias Biilmann",
      "ip":"127.0.0.1"
    },
    "created_at":"2013-09-12T00:55:39Z",
    "site_url":"http://synergy.bitballoon.com"
  }
]
```

Forms
=====

The `/forms` endpoint allow you to access forms from your BitBalloon sites. You can scope forms to a specific site with `/sites/{site_id}/forms`.

Get Forms
---------

* `GET /forms` will return a list of forms

```json
[
  {
    "id":"ac0865cc46440b1e64666f520e8d88d670c8a2f6",
    "site_id":"0d3a9d2f-ef94-4380-93df-27ee400e2048",
    "name":"Landing Page",
    "paths":["/index"],
    "submission_count":3,
    "fields": [
      {"name":"name","type":"text"},
      {"name":"email","type":"email"},
      {"name":"phone","type":"text"},
      {"name":"company","type":"text"},
      {"name":"website","type":"url"},
      {"name":"number_of_employees","type":"select"}
    ],
    "created_at":"2013-09-18T20:26:19Z"
  }
]
```

Files
=====

All files deployed by BitBalloon can be read, updated and deleted through the API. Where the public URL of a file will serve the processed version, the files accessed through the API are the original uploaded files. Any changes to a file will trigger a reprocessing of the site and a new deploy will be stored in the site history. This means all changes made through the API can always be rolled back by the user through the dashboard UI.

Get Files
---------

* `GET /sites/{site_id}/files` will return a list of all the files in the current deploy

```json
[
  {"id":"/index.html",
  "path":"/index.html",
  "sha":"20828dcdf2cd07e5980fe52759101591bf5014ab",
  "mime_type":"text/html",
  "size":27232
  }
]
```

Get File
--------

* `GET /sites/{site_id}/files/{path-to-file}` returns the file

```json
{"id":"/index.html",
"path":"/index.html",
"sha":"20828dcdf2cd07e5980fe52759101591bf5014ab",
"mime_type":"text/html",
"size":27232
}
```

You can get the raw contents of the file by using the custom media type `application/vnd.bitballoon.v1.raw` as the Content-Type of your HTTP request.

Put File
--------

* `PUT /sites/{site_id}/files/{path-to-file}` will add or update a file, reprocess all assets and create a new deploy

The request body will be used as the new content for this file. If the site is still in uploading mode (after creating a site with a list of files) and this is the last file 

Delete File
-----------

* `DELETE /sites/{site_id}/files/{path-to-file}` will delete a file from the site.

Note that this creates a new version of the site without the deleted file. The old version can still be rolled back and the file does not get deleted from BitBalloon's servers.

Snippets
========

Snippets are code snippets that will be injected into every HTML page of the website, either right before the closing head tag or just before the closing body tag. Each snippet can specify code for all pages and code that just get injected into "Thank you" pages shown after a successful form submissions.

Get Snippets
------------

* `GET /sites/{site_id}/snippets` get a list of snippets specific to this site (a reseller may set global snippets that won't be included in this list)

```json
[
  {
    "id":0,
    "title":"Test",
    "general":"\u003Cscript\u003Ealert(\"Hello\")\u003C/script\u003E",
    "general_position":"head",
    "goal":"",
    "goal_position":"footer"
  }
]
```

The `general` property is the code that will be injected right before either the head or body end tag. The `general_position` can be `head` or `footer` and determines whether to inject the code in the head element or before the closing body tag.

The `goal` property is the code that will be injected into the "Thank you" page after a form submission. `goal_position` determines where to inject this code.

Get Snippet
-----------

* `GET /sites/{site_id}/snippets/{snippet_id}` get a specific snippet

```json
{
  "id":0,
  "title":"Test",
  "general":"\u003Cscript\u003Ealert(\"Hello\")\u003C/script\u003E",
  "general_position":"head",
  "goal":"",
  "goal_position":"footer"
}
```

Add Snippet
--------------

* `POST /sites/{site_id}/snippets` add a new snippet to a site.

Update Snippet
--------------

* `PUT /sites/{site_id}/snippets/{snippet_id}` replace a snippet.

Delete Snippet
--------------

* `DELETE /sites/{site_id}/snippets/{snippet_id}` delete a snippet.
 
Metadata
========

Each site has a metadata object. The properties of the metadata object can be used within the snippets for a site by sing the [Liquid](https://github.com/Shopify/liquid) template syntax.

Get Metadata
------------

* `GET /sites/{site_id}/metadata` get the metadata for a site

```json
{
  "my_meta_key": "my_meta_value"
}
```

Update Metadata
---------------

* `PUT /sites/{site_id}/metadata` replace the metdata object with a new metadata object

Deploys
=======

You can access all deploys for a specific site.

Get Deploys
------------

* `GET /sites/{site_id}/deploys` a list of all deploys

```json
[
  {
    "id":"52465f435803544542000001",
    "premium":false,
    "claimed":true,
    "name":"synergy",
    "custom_domain":"www.example.com",
    "notification_email:"me@example.com",
    "url":"http://www.example.com",
    "deploy_url": "http://52465f435803544542000001.some-site.bitballoon.com",
    "admin_url":"https://www.bitballoon.com/sites/synergy",
    "screenshot_url":null,
    "created_at":"2013-09-17T05:13:08Z",
    "updated_at":"2013-09-17T05:13:19Z",
    "user_id":{"51f60d2d5803545326000005"},
    "state": "old"
  }
]
```

Get Deploy
----------

* `GET /sites/{site_id}/deploys/{deploy_id}` a specific deploy

```json
{
  "id":"52465f435803544542000001",
  "premium":false,
  "claimed":true,
  "name":"synergy",
  "custom_domain":"www.example.com",
  "notification_email:"me@example.com",
  "url":"http://www.example.com",
  "deploy_url": "http://52465f435803544542000001.some-site.bitballoon.com",
  "admin_url":"https://www.bitballoon.com/sites/synergy",
  "screenshot_url":null,
  "created_at":"2013-09-17T05:13:08Z",
  "updated_at":"2013-09-17T05:13:19Z",
  "user_id":{"51f60d2d5803545326000005"},
  "state": "old"
}
```

Restore Deploy
--------------

* `POST /sites/{site_id}/deploys/{deploy_id}/restore` restore an old deploy and make it the live version of the site

```json
{
  "id":"52465f435803544542000001",
  "premium":false,
  "claimed":true,
  "name":"synergy",
  "custom_domain":"www.example.com",
  "notification_email:"me@example.com",
  "url":"http://www.example.com",
  "deploy_url": "http://52465f435803544542000001.some-site.bitballoon.com",  
  "admin_url":"https://www.bitballoon.com/sites/synergy",
  "screenshot_url":null,
  "created_at":"2013-09-17T05:13:08Z",
  "updated_at":"2013-09-17T05:13:19Z",
  "user_id":{"51f60d2d5803545326000005"},
  "state": "current"
}
```

Users
=====

Mainly useful for reseller admins. Lets you access all users under your reseller account.

Get Users
---------

* `GET /users` get a list users you have access to

```json
[
  {
    "id": "52465eb05803543960000015",
    "email":"user@example.com",
    "affiliate_id":"",
    "site_count":1,
    "created_at":"2013-09-28T04:44:32Z",
    "last_login":"2013-09-28T04:44:33Z"
  }
]
```

Once you have a user id you can query sites, forms and submissions scoped to that user (ie. /users/{user_id}/sites).
