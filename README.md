# Part 06a: Authentication and Authorization  

This tutorial follows after: 
[Part 05: Implementing CRUD operations: Using Forms and POST requests](https://github.com/atcs-wang/inventory-webapp-05-handling-forms-post-crud)

You may have optionally done [Part 06b: First Deployment](https://github.com/atcs-wang/inventory-webapp-06-first-deployment) before this tutorial as well.

> Some more detailed "next steps" for using Auth0 w/ Node/Express here: [https://github.com/auth0/express-openid-connect/blob/master/EXAMPLES.md#2-require-authentication-for-specific-routes](https://github.com/auth0/express-openid-connect/blob/master/EXAMPLES.md#2-require-authentication-for-specific-routes)

# (6.0) Get started with Auth0

Go to [https://auth0.com/](https://auth0.com/) and create an account.

Once logged into, you can follow the `Getting Started -> Create Application` wizard. 
Choose:
  * At  `Create Application`, give your app a Name and choose `Regular Web Applications` as your application type.

  * Click on Settings.
    
  * `Allowed Callback URL` box should contain `http://localhost:3000/callback` 
  * `Allowed Logout URLs` box should contain `http://localhost:3000`. If necessary, change the numbers from `3000` to whatever your PORT value is during development.   

In your terminal, run the following command to install the necessary library
```js
npm install express express-openid-connect --save
```

Add the following lines to your .env file:
```js
AUTH0_SECRET= // TODO: Copy from Settings page in auth0
AUTH0_ISSUER_BASE_URL= // TODO: Copy from Settings page in auth0
APP_BASE_URL=http://localhost:3000
AUTH0_CLIENT_ID= // TODO: Copy from Settings page in auth0
```
  * Add the following code to your app.js file.
```js
// Auth0 configuration - pick values from environment variables
const authConfig = {
    authRequired: false, // change to true if you want all routes protected by default
    auth0Logout: true,
    secret: process.env.AUTH0_SECRET || 'change_this_secret',
    baseURL: process.env.AUTH0_BASE_URL || `http://localhost:${port}`,
    clientID: process.env.AUTH0_CLIENT_ID,
    issuerBaseURL: process.env.AUTH0_ISSUER_BASE_URL,

};

// Mount the auth router which adds /login, /logout, /callback
app.use(auth(authConfig));

// expose authentication status and user to all views
app.use((req, res, next) => {
    res.locals.isAuthenticated = req.oidc && req.oidc.isAuthenticated && req.oidc.isAuthenticated();
    res.locals.user = req.oidc && req.oidc.user ? req.oidc.user : null;
    next();
});
```
  * Restart your server and visit the `/` route, you should be prompted to log in.
  * Visit the `/me` route, to see the data available about the logged in user.
  * You can now use the email address of the logged in user as a parameter in your queries to personalized your site!

Optional: 
## (6.3) Adding a log-in / log-out button (+ nav bar)

We obviously need a way to log in and out that's more intuitive for the user than changing the URL manually. 

The natural solution is to add Login and Logout buttons (that is, links to `/login` and `/logout`) to the navbar at the top of our pages 
```html
    <a href="/login" class="btn blue">Login</a>
    <a href="/logout" class="btn red">Logout</a>
```

Go ahead and update the `<nav>` in each of the EJS views, adding the buttons in both the main nav, and the mobile side nav. 

```html
...
<!-- Nav bar -->
<header>
    <nav>
        <div class="nav-wrapper">
            <a href="/" class="brand-logo"><i class="material-icons left">school</i>Homework Manager</a>
            <a href="#" data-target="mobile-nav" class="sidenav-trigger"><i class="material-icons">menu</i></a>
            <!-- Main nav, hidden for small screens -->
            <ul id="desktop-nav" class="right hide-on-med-and-down">
                <li><a href="/"><i class="material-icons left">home</i> Home</a></li>
                <li><a href="/assignments"><i class="material-icons left">list</i> Assignments</a></li>
                <li><a href="/login" class="btn blue">Login</a></li>
                <li><a href="/logout" class="btn red">Logout</a></li>
            </ul>
        </div>
    </nav>
    <!-- Mobile nav menu, shown when menu button clicked -->
    <ul id="mobile-nav" class="sidenav">
        <li><a href="/"><i class="material-icons left">home</i> Home</a></li>
        <li><a href="/assignments"><i class="material-icons left">list</i> Assignments</a></li>
        <li><a href="/login" class="btn blue">Login</a></li>
        <li><a href="/logout" class="btn red">Logout</a></li>
    </ul>
</header>
```


Test the buttons to make sure they work.

## (6.4) Changing view on authentication; displaying user information

We probably only want to show one of the two buttons at a time - LOGIN if not yet authenticated, and LOGOUT if already authenticated. We might also like to show, if you are logged in, some of your user information, making it clear who you're logged in as.

The `/authtest` route from above demonstrated the `req.oidc.isAuthenticated()` method which tells us whether we are logged in, and the `/profile` route demonstrated the `req.oidc.user` object, which has user info included with the auth token.

In general, the state of an incoming request's authentication is available via `req.oidc`, and that can be used to determine how we respond. We want our server to make the state of authentication available to EJS during the rendering step. 


One way is pass it as part of the context parameter given at `res.render(...)`.

For example we *could* (but don't actually) change the `res.render` in the `/assignments` route handler to something like this:

```js
app.get( "/assignments", ( req, res ) => {
    ...
            res.render('assignments', { assignment list : results , 
                                  isLoggedIn : res.oidc.isAuthenticated()
                                  user: res.oidc.user });
    ...
} );
```

This would provide `assignments.ejs` the `isLoggedIn` and `user` variables, and those could be used to change the view. 

However, if we wanted that information for *every* page we render, we'd have to repeatedly pass the value to each and every `res.render()` call - tedious, and easy to miss one as our app grows.

A better way is to create a single middleware that, for all routes, will add a "local" property to the `res` response object, passing the information to EJS for all page renders.

Add this to the `app.js` after the other middleware (`app.use`) but before any route handlers (`app.get`, `app.post`, etc.)

```js
app.use((req, res, next) => {
    res.locals.isLoggedIn = req.oidc.isAuthenticated();
    res.locals.user = req.oidc.user;
    next();
})
```

Now, we can assume the availability of two variables in all of our `.ejs` files:
* `isLoggedIn`, a boolean indicating our authentication status
* `user`, which is either `undefined` if not logged in, or the user info object from the auth token. 

Let's use them to customize our nav bar view ( on every page):

```html
<header>
         <nav>
            <div class="nav-wrapper">
                <a href="/" class="brand-logo"><i class="material-icons left">school</i>Homework Manager</a>
                <a href="#" data-target="mobile-nav" class="sidenav-trigger"><i class="material-icons">menu</i></a>
                <ul id="desktop-nav" class="right hide-on-med-and-down">
                    <li><a href="/"><i class="material-icons left">home</i> Home</a></li>
                    <li><a href="/assignments"><i class="material-icons left">list</i> Assignments</a></li>
                    <% if (isLoggedIn) { %>
                        <li><a href="/profile"><i class="material-icons left">person</i> Hello, <%=user.name%></a> </li>
                        <li><a href="/logout" class="btn red">Logout</a></li>                        
                    <% } else { %>
                        <li><a href="/login" class="btn blue">Login</a></li>
                    <% } %>
                </ul>
            </div>
        </nav>
        <ul id="mobile-nav" class="sidenav">
            <li><a href="/"><i class="material-icons left">home</i> Home</a></li>
            <li><a href="/assignments"><i class="material-icons left">list</i> Assignments</a></li>
            <% if (isLoggedIn) { %>
                <li><a href="/profile"><i class="material-icons left">person</i> Hello, <%=user.name%></a> </li>
                <li><a href="/logout" class="btn red">Logout</a></li>                        
            <% } else { %>
                <li><a href="/login" class="btn blue">Login</a></li>
            <% } %>
        </ul>
    </header>
```

Test your nav bar on each page, both logged in and out. 

## (6.5) Authorization for specific routes
Now that our users can log in/out and see their authentication status, we now move the issue of *authorization* - only allowing certain kinds of people into certain parts of the site.

The most basic kind of authorization simply requires that the users have been authenticated at all. Auth0's `express-openid-connect` library provides the `requiresAuth()` function for this purpose, demonstrated by the sample `/profile` route we set up earlier:

```js
    app.get('/profile', requiresAuth(), (req, res) => {
      res.send(JSON.stringify(req.oidc.user));
    });
```

`requiresAuth()` produces a middleware function, which can be applied as a sort of "pre-handler" to any route we want to restrict to just authenticated users.

The only route we really want available to un-authenticated users is the homepage (plus the `/authtest` route); the rest should require authentication.

We can change each of these routes
```js
app.get( "/assignments", ( req, res ) => { ... }
app.get( "/assignments/:id", ( req, res ) => { ... }
app.get("/assignments/:id/delete", ( req, res ) => { ... }
app.post("/assignments/:id", ( req, res ) => { ... }
app.post("/assignments", ( req, res ) => { ... }
```

to include `requiresAuth()` as a route-specific middleware that occurs before the handlers, like this:

```js
app.get( "/assignments", requiresAuth(), ( req, res ) => { ... }
app.get( "/assignments/:id", requiresAuth(), ( req, res ) => { ... }
app.get("/assignments/:id/delete", requiresAuth(), ( req, res ) => { ... }
app.post("/assignments/:id", requiresAuth(), ( req, res ) => { ... }
app.post("/assignments", requiresAuth(), ( req, res ) => { ... }
```

Now, log out and attempt accessing various pages. You should be, in each case, redirected to the Auth0 login page.

> NOTE ON ALTERNATIVES: We could have also used the `requiresAuth()` middleware the way we have other middleware, like this:
> ```js 
> app.use(requiresAuth());
> ```
> This would require authentication for ALL route handlers below this line;  any route handlers above this line will not require authentication.

> If you simply want to require authentication for *every* page,  yet another alternative would be to change the `config` object's `authRequired` property to `true`.

## (6.6) Associating assignment data with users
Now that you can only access certain routes when authenticated, we can confidently utilize their user information in those routes. 

Our final step is a very consequential one: associate every assignment in our database with the user who created and owns that assignment. 

The identification token (`req.oidc.user`) always has a field called `sub` (short for "subject") which is a unique identifier for any user (max 255 characters long). So we can use `req.oidc.user.`sub`` and 

> Another option is to use the user's `sub` (`req.oidc.user.`sub``) as a unique identifier. Most social logins are ultimately based on an `sub` address (Google for example), but its not universal unless you're selective about your login options.


### (6.6.1) Add a `userId` column to the database table

First, we need to add a column to the database table called something like "userId".


If you're using it, we can update `db/db_create.js` appropriately so the (re-)created `assignment` table has that column. 

My `CREATE TABLE assignments` script changed from:
  ```sql
    CREATE TABLE assignments (
        assignmentId INT NOT NULL AUTO_INCREMENT,
        title VARCHAR(45) NOT NULL,
        priority INT NULL,
        subjectId INT NOT NULL,
        dueDate DATE NULL,
        description VARCHAR(150) NULL,
        PRIMARY KEY (assignmentId),
        INDEX assignmentSubject_idx (subjectId ASC),
        CONSTRAINT assignmentSubject
            FOREIGN KEY (subjectId)
            REFERENCES subjects (subjectId)
            ON DELETE RESTRICT
            ON UPDATE CASCADE);
  ```
  to 

  ```sql
          CREATE TABLE assignments (
        assignmentId INT NOT NULL AUTO_INCREMENT,
        title VARCHAR(45) NOT NULL,
        priority INT NULL,
        subjectId INT NOT NULL,
        dueDate DATE NULL,
        description VARCHAR(150) NULL,
        userId VARCHAR(255) NULL, -- this is the new line!
        PRIMARY KEY (assignmentId),
        INDEX assignmentSubject_idx (subjectId ASC),
        CONSTRAINT assignmentSubject
            FOREIGN KEY (subjectId)
            REFERENCES subjects (subjectId)
            ON DELETE RESTRICT
            ON UPDATE CASCADE);
  ```

Then re-create your database (and repopulate it) with either:
```
> node db/db_create.js
> node db/db_insert_sample.js
```
or (if you set up the `npm` script in `package.json`)
```
> npm run dbcreate
> npm run dbsample
```

> You may have noticed that our `db_insert_sample.js` populates `assignments` with some sample rows , but does not set a `userId` for any of them. As a result, they don't belong to anyone! So, we could, if we wanted, remove part that from our script. 
>
> We DO still need (for now) the part of `db_insert_sample.js` that fills in the `subjects` table.

### (6.6.2) Update the CREATE operation

First, we need to make it such that whenever a user tries to create a new assignment, their `sub` is entered into the database along with it.

Find the SQL statement that inserts a new assignment in `app.js`. From this:

```sql
INSERT INTO assignments 
    (title, priority, subjectId, dueDate) 
VALUES 
    (?, ?, ?, ?);
```

to this:
```sql
INSERT INTO assignments 
    (title, priority, subjectId, dueDate, userId) 
VALUES 
    (?, ?, ?, ?, ?);
```

Then, in the `/assignments` POST request handler that executes that SQL, we can provide the user's `sub` from the request as the final `?`. Change from this:

```js
app.post("/assignments", requiresAuth(), ( req, res ) => {
    db.execute(create_assignment_sql, [req.body.title, req.body.priority, req.body.subject, req.body.dueDate], (error, results) => {

      ...
    }
}
```

to this:

```js
app.post("/assignments", requiresAuth(), ( req, res ) => {
    db.execute(create_assignment_sql, [req.body.title, req.body.priority, req.body.subject, req.body.dueDate, req.oidc.user.sub], (error, results) => {

      ...
    }
}
```

Try using the "Add" form. If you check your database, you ought to see that whatever table row you added includes the logged-in user's `sub`.

### (6.6.2) Update the READ (assignment list) operation

Now we need to restrict our users to only seeing data that they've created and are associated with.

With the SQL that selects the full list of assignments, change it from :

```sql 
    SELECT 
        assignmentId, title, priority, subjectName, 
        assignments.subjectId as subjectId,
        DATE_FORMAT(dueDate, "%m/%d/%Y (%W)") AS dueDateFormatted
    FROM assignments
    JOIN subjects
        ON assignments.subjectId = subjects.subjectId
    ORDER BY assignments.assignmentId DESC
```

to :
```sql
    SELECT 
        assignmentId, title, priority, subjectName, 
        assignments.subjectId as subjectId,
        DATE_FORMAT(dueDate, "%m/%d/%Y (%W)") AS dueDateFormatted
    FROM assignments
    JOIN subjects
        ON assignments.subjectId = subjects.subjectId
    WHERE assignments.userId = ?    --this is the new line
    ORDER BY assignments.assignmentId DESC
```

Then, in the `/assignments` GET request handler that executes that SQL, we can once again provide the user's `sub` from the request as the new `?`. Change from this:


```js
app.get( "/assignments", requiresAuth(), ( req, res ) => {
    db.execute(read_assignments_all_sql, (error, results) => {
      ...
    }
}
```

to this:

```js
app.get( "/assignments", requiresAuth(), ( req, res ) => {
    db.execute(read_assignments_all_sql, [req.oidc.user.sub], (error, results) => {

```

Now, revisit the `/assignments` page. You should only see the assignment(s) created after `userId` was incorporated into the database. Add more assignments and confirm that they appear. Log out and log in as various users, creating new assignments and confirming that only those assignments, and not other user's data, is displayed.

### (6.6.3) Update the UPDATE, DELETE, and READ (assignment) operations

While users can now only see data that they've created themselves on the `/assignments` assignment list page, they can still access the assignment detail pages for assignments created by *any* user. 

> Try navigating to `/assignment/:id` for various ids of assignments that aren't created by the logged in user to see that this is the case.

Even worse, they can also edit and delete those assignments as well!

> Try both the edit form and delete buttons on those pages.

We need to restrict the accessibility of the routes associated with the UPDATE, DELETE, and READ operations for individual assignments. Fortunately, all are quite similar.


#### Restrict UPDATE:

With the SQL that updates a single assignment, change it from :

```sql 
UPDATE
    assignments
SET
    title = ?,
    priority = ?,
    subjectId = ?,
    dueDate = ?,
    description = ?
WHERE
    assignmentId = ?
```

to :
```sql
UPDATE
    assignments
SET
    title = ?,
    priority = ?,
    subjectId = ?,
    dueDate = ?,
    description = ?
WHERE
    assignmentId = ?
AND userId = ?  -- this is the new line
```

Then, in the `/assignment/:id` POST request handler that executes that SQL, we can provide the user's `sub` from the request as the new `?`. Change from this:

```js
app.post("/assignment/:id", requiresAuth(), ( req, res ) => {
    db.execute(update_assignment_sql, [req.body.title, req.body.priority, req.body.subject, req.body.dueDate, req.body.description, req.params.id], (error, results) => {
      ...
    }
}
```

to this:

```js
app.post("/assignment/:id", requiresAuth(), ( req, res ) => {
    db.execute(update_assignment_sql, [req.body.title, req.body.priority, req.body.subject, req.body.dueDate, req.body.description, req.params.id, req.oidc.user.sub], (error, results) => {
      ...
    }
}
```
> Check that filling out the Edit form for an assignment owned by another user results in no changes. It should still refresh the page on submit, but no changes will be made.

#### Restrict DELETE 
Next, the `DELETE` operation needs to be locked down. 

With the SQL that deletes a single assignment, change it from :

```sql 
    DELETE 
    FROM
        assignments
    WHERE
        assignmentId = ?
```

to :
```sql
    DELETE 
    FROM
        assignments
    WHERE
        assignmentId = ?
    AND userId = ?  -- this is the new line
```

Then, in the `/assignment/:id/delete` GET request handler that executes that SQL, we can once again provide the user's `sub` from the request as the new `?`. Change from this:


```js
app.get("/assignment/:id/delete", requiresAuth(), ( req, res ) => {
    db.execute(delete_assignment_sql, [req.params.id], (error, results) => {
      ...
    }
}
```

to this:
```js
app.get("/assignment/:id/delete", requiresAuth(), ( req, res ) => {
    db.execute(delete_assignment_sql, [req.params.id, req.oidc.user.sub], (error, results) => {
      ...
    }
}
```

> Verify that attempting a delete of another users' assignment  via the button does not, in fact, cause the deletion of the assignment; it should redirect you to the `/assignments` page, but the database should not have changed, and the assignment detail page still be accessible. 


#### Restrict READ (single assignment)

Lastly, with the SQL that selects a single assignment, change it from :
```sql 
    SELECT
        assignmentId, title, priority, subjectName,
        assignments.subjectId as subjectId,
        DATE_FORMAT(dueDate, "%W, %M %D %Y") AS dueDateExtended, 
        DATE_FORMAT(dueDate, "%Y-%m-%d") AS dueDateYMD, 
        description
    FROM assignments
    JOIN subjects
        ON assignments.subjectId = subjects.subjectId
    WHERE assignmentId = ?
```

to :
```sql
    SELECT
        assignmentId, title, priority, subjectName,
        assignments.subjectId as subjectId,
        DATE_FORMAT(dueDate, "%W, %M %D %Y") AS dueDateExtended, 
        DATE_FORMAT(dueDate, "%Y-%m-%d") AS dueDateYMD, 
        description
    FROM assignments
    JOIN subjects
        ON assignments.subjectId = subjects.subjectId
    WHERE assignmentId = ?
    AND assignments.userId = ? --this is the new line
```

Then, in the `/assignment/:id` GET request handler that executes that SQL, we can once again provide the user's `sub` from the request as the new `?`. Change from this:


```js
app.get( "/assignment/:id", requiresAuth(), ( req, res ) => {
    db.execute(read_assignment_detail_sql, [req.params.id], (error, results) => {
      ...
    }
}
```

to this:

```js
app.get( "/assignment/:id", requiresAuth(), ( req, res ) => {
    db.execute(read_assignment_detail_sql, [req.params.id, req.oidc.user.sub], (error, results) => {
      ...
    }
}
```

Now, revisit various `/assignment/:id` pages, attempting to access assignments not created by the current user. You should get a `404` response.

> Although it's not exactly true that the assignment doesn't exist, it is appropriate to act like it doesn't since the user should not be aware of the existence of other user's assignments. More accurately, we'd say that it is not authorized for that user to access that page. In such cases, we sometimes prefer a `403 Forbidden` code instead of `404 Not Found`.

> Now both the `Edit` form and `Delete` button cannot be directly accessed any longer by unauthorized users.  It may seem unnecessary to have restricted those other operations when restricting access to the list or detail page makes it pretty difficult for normal users to accidentally edit or delete an assignment they can't see. But it's safer to be thorough, as arbitrary GET and POST requests can in fact still be made by malicious users without forms or buttons. 

### (6.7) Conclusion

At this point, we have integrated authentication and some basic authorization.  We have associated data with specific users, and only authorized users to perform CRUD operations on their own data.

What next? If you haven't yet, we are in a position to deploy our app for the first time, hosting it publically on the web. This is optional, but it is a good experience to have. There are just a few minor changes to make before we do so.

Check out [# Part 06b: First Deployment](https://github.com/atcs-wang/inventory-webapp-06-first-deployment)

However, our app currently restricts users to a fixed set of subjects to choose from; it would be great if users could create and manage their own custom subjects. As we do that, however, there are some structural changes and improvements that we ought to make to our project that will pay dividends down the road for future expansion.

Check out [Part 07: More CRUD operations, code reorganization](https://github.com/atcs-wang/inventory-webapp-07-reorganization-expansion). Note that the README tutorial is currently incomplete.


