Reviewing the developer's claim

The claim is wrong. The button being visible only to editors means nothing on the server side. Anyone can open browser dev tools, find the request the button sends, and replay it directly with curl or Postman. The button is a UI convenience, not a security boundary.

The endpoint accepts an article ID and a new status. If the server doesn't check who is calling it and whether that person is allowed to change that article, then anyone can change any article's status. That's the whole problem.

Server-side checks I would require

       1. Authentication. Is the request from a logged-in user? If not, reject with 401.

       2. Capability check. Does the authenticated user have permission to edit this specific post? In WordPress, current_user_can('edit_post', $post_id). If not, reject with 403.

       3. Nonce verification. Was the request sent with a valid WordPress nonce for this action? This protects against CSRF. If not, reject with 403.

       4. Input validation. Is the article ID a positive integer? Is the status one of the allowed values? If not, reject    with 400.

       5. Post existence check. Does a post with that ID exist? If not, reject with 404.

       6. Status transition validity. Is the transition allowed for this user? An editor without publish_posts should not  move a draft to publish. If not, reject with 403.

       7. Rate limiting. Reasonable protection against hammering the endpoint. Optional but recommended.

Only after all seven pass does the update run.

Pseudocode, in order
text

      function update_article_status(request):
       user = get_current_user()
         if user is null:
           return 401 Unauthorized

       post_id = request.body.post_id
        if not current_user_can('edit_post', post_id):
          return 403 Forbidden

       nonce = request.headers['X-WP-Nonce']
       if not wp_verify_nonce(nonce, 'update_article_status'):
           return 403 Forbidden

        if not is_positive_integer(post_id):
            return 400 Bad Request

        new_status = request.body.status
        allowed = ['draft', 'pending', 'publish', 'private']
        if new_status not in allowed:
            return 400 Bad Request

        post = get_post(post_id)
        if post is null:
            return 404 Not Found

        if new_status == 'publish' and not current_user_can('publish_posts'):
            return 403 Forbidden

        if rate_limit_exceeded(user.id, 'update_article_status'):
            return 429 Too Many Requests

        wp_update_post({ 'ID': post_id, 'post_status': new_status })
        log_event(user.id, post_id, new_status)
        return 200 OK

Two tests that should be rejected

Test 1 — Replay without authentication. Take the request the button sends, strip the auth cookie, send it again with curl. Expected: 401 Unauthorized. If it returns 200, the endpoint is broken.
Test 2 — Editor publishing without permission. Log in as an Editor without publish_posts capability. Send a request to move a draft to publish. Expected: 403 Forbidden. If it returns 200, the capability check is missing or too loose.

Two more I would add:

Test 3 — Invalid post ID. Send post_id: "abc" or post_id: -1. Expected: 400 Bad Request.
Test 4 — CSRF without nonce. Send a valid authenticated request with no nonce header. Expected: 403 Forbidden.

How I'd raise it with the developer

Privately, in writing, framed as a shared problem, not a personal failing:

Hey, I was reviewing the article status endpoint and I think there's a gap. The button is editor only in the UI, but the endpoint itself doesn't verify who's calling it. I tested it with curl and it accepted a request from a non-authenticated user. Screenshot attached. This means anyone who finds the URL can change article statuses. Can we add the capability check and nonce verification before this ships? I'm happy to pair on it if that helps.

Direct, evidence-based, not accusatory. The developer sees proof, not a lecture.

What must be fixed before release

     1. Authentication check

     2. Capability check on the specific post

     3. Nonce verification

     4. Input validation on post ID and status

     5. Post existence check

     6. Block capability-restricted transitions

     7. Rate limiting as a safety net

All seven before release. None optional, but 1, 2, and 3 turn the endpoint from open to locked
