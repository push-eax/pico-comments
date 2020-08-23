# pico-comments

This plugin implements comments for Pico CMS.

I wrote this for my own personal use, but felt that others might get some use out of it because a comments plugin for Pico did not exist yet.

Features
---
- Comments stored in a similar manner to pages
- Comment replies
- Comment review
- Comment size limits
- Antispam using a honeypot form field
- IP address logging
- Theoretically scalable

Usage and configuration
---
To use this plugin, you need to:
1. Add PicoComments.php to your Pico plugins directory
2. Configure and enable it in your config.yml
3. Add comment submission forms and rendering macros to your theme

The plugin is configured in the site config.yml like so:
```
PicoComments:
    # whether the comments plugin is enabled. comment data will not be available to Twig if this is false
    enabled: true
    # maximum character size of comments. comments larger than this will be rejected
    comment_size_limit: 40000
    # whether comments must be approved before displaying them to other users
    comment_review: false
```

Comment submission can be enabled/disabled for each page in that page's frontmatter:
```
---
comments: true
---
```

If ```comments``` is set to ```false```, existing comments on that page will still be available to Twig for rendering, but no new comments will be accepted.

Comment data and comment submission success/error messages are exposed as Twig variables. Please see comments.twig for an example comment submission form and comment rendering code.

Comments that are submitted while ```comment_review``` is enabled in the site config are not exposed to Twig. To approve a comment that is under review, open it with a text editor and change the "pending" field from "true" to "false". To deny it, leave "pending" as "true" or delete the comment file from disk.

Technical details
---
Comments are created by submitting a POST request to the URL of a page with comments enabled. The plugin looks for these parameters:
- comment_author
- comment_content
- comment_replyguid must be included if the comment is a reply to another comment

All other comment properties (guid, date of posting, ip address, pending review) are generated on the server. 

Comment content and author names are sanitized before being saved. Comment reply GUIDs are validated before the comment is saved, and if the comment pointed to by the reply GUID does not exist, the comment will not be accepted.

Comments are stored as files in the plugin's ```content``` directory in a manner very similar to Pico's own page structure. The structure of a comment file is as follows:
```
---
guid
guid of comment being replied to, if this comment is in reply to another comment
date posted, in UNIX time
ip address of author
author name
pending status
---
content
```

Files are named using the comment guid followed by a ```.md``` extension, though I chose not to render the comment content as Markdown to avoid cross-site scripting issues.

Comment storage design
---
I chose to store comments on disk as a parent-pointer tree, because a child-pointer representation would require comments to maintain a list of children which is updated each time the comment is replied to, which is not necessary with a parent-pointer tree. However, a parent-pointer tree is much harder to recurse from the top down, which is what we want when rendering the comments; creating a parent comment as a div and recursing downward to create its children inside of it is much simpler than recursing upward and storing child comments in memory until we reach the top of the tree and then rendering downwards somehow (this is the sort of thing that should **not** be done in Twig). We achieve the best of both schemes by storing comments as parent-pointer and exposing them to Twig as child-pointer, making it easy for Twig to render replies recursively without complex logic. Using both tree types necessitates some means of transforming one to the other, which is the core logic of this plugin.

First, comments are read from disk into memory. Each comment is stored as a dictionary of its frontmatter (meta) keys to the values corresponding to them, plus a "content" key that contains its content:
```
Array
(
    [guid] => ce7b48f8ce53dff6af8a3f4b5b9261d6
    [reply_guid] => d3b964839536a5fdb1006e6f9d9140e2
    [date] => 2020-08-20 15:25:27
    [ip] => 127.0.0.1
    [author] => test5
    [time] => 1597951527
    [date_formatted] => August 08, 2020 15:25
    [content] => Hello, world!
)
```

A PHP dictionary is used to store comments: comment GUIDs as keys, arrays which contain children of that comment as values. If a comment has a parent, it's placed in the array corresponding to that parent's GUID. If it doesn't have a parent (top level comment), it's placed in an array with a null key ([""]).

Once we have established the comment structure in memory, we transform it. The transformation algorithm works like this:
```
Given an array of comments (starting with the array of top level comments corresponding to the null key), for each comment in the array:
    If this comment's GUID is a key in the dictionary (it has children):
        Recurse into the array corresponding to that GUID (recursion happens first so that children are fully populated with their own descendants before moving them)
        Set this comment's replies array equal to that array of children, making a copy inside this comment
        Sort this comment's replies array by date
        Remove array of children from the dictionary now that it is contained within this comment
```

Once the tree is transformed, we expose it to Twig as the ```comments``` variable, which can be recursed using a Twig macro to render it. We can access the total number of comments in Twig using the ```comments_number``` variable.

The use of a dictionary to build the child-pointer tree eliminates any need for searching, so this plugin should scale to large numbers of comments, but I haven't tested it with anything more than a few dozen comments.

Issues, assumptions, future ideas
---
- Comments are stored in a hierarchy that mirrors the structure of the site. If the site structure changes, the comments structure needs to be changed to match. This could be addressed by changing the design to store the page ID within each comment and all comments in a single directory, but I decided against that, instead keeping comments for each page in a directory corresponding to that page, which improves performance by limiting the tree-building loops only to the comments on the current page.
- There is no captcha, but the current antispam solution should suffice unless someone writes a bot specifically to ignore the honeypot form field. IP addresses are stored, so IP-based antispam would be straightforward to implement.
- Users cannot delete comments, because no user authentication is implemented. A simple approach might be to use PHP sessions to allow users to delete their comments as long as their session is valid, but not after their session cookie expires and they are no longer identifiable.
- The plugin responds to a POST request directly with a newly rendered page rather than issuing a redirect to the same page. This means that if the user refreshes the page after submitting a comment, they may submit a duplicate comment. I chose to do this because submission messages (success/error, etc.) are exposed as Twig variables, which means they will not survive a redirect since they exist on a per-request basis. Once again, sessions could address this.
- There is some code duplication between the print_comments macro and the top-level submission form. This could be resolved with another macro.

Contributing
---
I welcome suggestions, comments, and code. 
- If you feel your contribution would improve this plugin for the general use case, please submit a pull request. 
- If you find a bug, open an issue and describe what's happening. Please include verbose PHP and web server logs and screenshots of the issue, where applicable.

I don't have a lot of time to continue development of this plugin except for my own purposes. I can't guarantee that I will fix issues or implement new features promptly, or at all. If I do, and if I believe such improvements would be useful to users, I will release them.
