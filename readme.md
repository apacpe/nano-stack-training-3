# Node.js Web Application - Part 3
This is a continuation from [this guide](https://github.com/apacpe/nano-stack-training-2). We will be further modifying the app to allow integration of the app to a HubSpot portal via OAuth. Make sure you have a [HubSpot developer portal](https://developers.hubspot.com/) before you start on this.

In this guide, we will be achieving these 2 items:
1. Configure HubSpot OAuth logic into the app
2. Incorporate [HubSpot contacts search API](https://developers.hubspot.com/docs/methods/contacts/search_contacts) and use OAuth access token as authentication for the request

### Install node modules
Install these node packages:
1. `npm install node-cache`
2. `npm install express-session`
3. `npm install request`
4. `npm install request-promise-native`

### Create a `.env` file in the root of your project directory with your developer portal app's client ID and client secret
1. Create a `.env` file using `touch .env`
2. Insert the client ID, secret and scope in the file in this format:
```
CLIENT_ID='xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
CLIENT_SECRET='yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy'
SCOPE='crm.objects.contacts.write,forms'
```

### Set up required node modules and variables
Add the following codes to the top of your app.js file:
```
require('dotenv').config();
const request = require('request-promise-native');
const NodeCache = require('node-cache');
const session = require('express-session');

const CLIENT_ID = process.env.CLIENT_ID;
const CLIENT_SECRET = process.env.CLIENT_SECRET;
// Supports a list of scopes as a string delimited by ',' or ' ' or '%20'
const SCOPES = (process.env.SCOPE.split(/ |, ?|%20/) || ['crm.objects.contacts.write']).join(' ');

const REDIRECT_URI = `http://localhost:${PORT}/oauth-callback`;

const refreshTokenStore = {};
const accessTokenCache = new NodeCache({ deleteOnExpire: true });

// Use a session to keep track of client ID
app.use(session({
  secret: Math.random().toString(36).substring(2),
  resave: true,
  saveUninitialized: true,
  cookie: {
  	maxAge: 12 * 30 * 24 * 60 * 60 * 1000
  }
}));
```

### Create routes to initiate and complete OAuth authorization
1. Create a variable that stores the OAuth authorization url:
```
const authUrl =
  'https://app.hubspot.com/oauth/authorize' +
  `?client_id=${encodeURIComponent(CLIENT_ID)}` + // app's client ID
  `&scope=${encodeURIComponent(SCOPES)}` + // scopes being requested by the app
  `&redirect_uri=${encodeURIComponent(REDIRECT_URI)}`; // where to send the user after the consent page
```
2. Create a get route handler to redirect users to the authorization url:
```
app.get('/install', (req, res) => {
  console.log('Initiating OAuth 2.0 flow with HubSpot');
  console.log("Step 1: Redirecting user to HubSpot's OAuth 2.0 server");
  res.redirect(authUrl);
  console.log('Step 2: User is being prompted for consent by HubSpot');
});
```
3. Create a get route handler to obtain the code from the redirected uri and exchange it for tokens:
```
app.get('/oauth-callback', async (req, res) => {
  console.log('Step 3: Handling the request sent by the server');

  // Received a user authorization code, so now combine that with the other
  // required values and exchange both for an access token and a refresh token
  if (req.query.code) {
    console.log('  > Received an authorization token');

    const authCodeProof = {
      grant_type: 'authorization_code',
      client_id: CLIENT_ID,
      client_secret: CLIENT_SECRET,
      redirect_uri: REDIRECT_URI,
      code: req.query.code
    };

    // Step 4
    // Exchange the authorization code for an access token and refresh token
    console.log('Step 4: Exchanging authorization code for an access token and refresh token');
    const token = await exchangeForTokens(req.sessionID, authCodeProof);
    if (token.message) {
      return res.redirect(`/error?msg=${token.message}`);
    }
    console.log(req.sessionID);
    // Once the tokens have been retrieved, use them to make a query
    // to the HubSpot API
    res.redirect(`/admin`);
  }
});
```

### Create helper functions for exchanging and refreshing tokens
1. Create function to perform post request to get access and refresh tokens:
```
const exchangeForTokens = async (userId, exchangeProof) => {
  try {
    const responseBody = await request.post('https://api.hubapi.com/oauth/v1/token', {
      form: exchangeProof
    });
    // Usually, this token data should be persisted in a database and associated with
    // a user identity.
    const tokens = JSON.parse(responseBody);
    refreshTokenStore[userId] = tokens.refresh_token;
    accessTokenCache.set(userId, tokens.access_token, Math.round(tokens.expires_in * 0.75));

    console.log('  > Received an access token and refresh token');
    return tokens.access_token;
  } catch (e) {
    console.error(`  > Error exchanging ${exchangeProof.grant_type} for access token`);
    return JSON.parse(e.response.body);
  }
};
```
2. Create function to store and pass required parameters for the token post request: 
```
const refreshAccessToken = async (userId) => {
  const refreshTokenProof = {
    grant_type: 'refresh_token',
    client_id: CLIENT_ID,
    client_secret: CLIENT_SECRET,
    redirect_uri: REDIRECT_URI,
    refresh_token: refreshTokenStore[userId]
  };
  return await exchangeForTokens(userId, refreshTokenProof);
};
```
3. Create function to get access tokens stored in node cache module or initiate process to refresh access token:
```
const getAccessToken = async (userId) => {
  // If the access token has expired, retrieve
  // a new one using the refresh token
  if (!accessTokenCache.get(userId)) {
    console.log('Refreshing expired access token');
    await refreshAccessToken(userId);
  }
  return accessTokenCache.get(userId);
};

const isAuthorized = (userId) => {
  return refreshTokenStore[userId] ? true : false;
};
```

### Create EJS templates for the new install and admin routes
1. Create a page for visitors to install the app to their HubSpot portal (i.e. adminInstall.ejs):
```
<html>
 <body>
	<div>
	 <a href="/install">Install app to HubSpot here</a>
	</div>
 </body>
</html>
```
2. Create a page for visitors who have installed the app to search for HubSpot contacts (i.e. admin.ejs):
```
<html>
 <body>
  <h1>Search for contacts in HubSpot</h1>
	 <div class="contactsearch">
	  <form action="/admin" method="post">
		 <input type="text" name="searchinput">
		 <input type="submit" name="search" value="Search for contact">
		</form>
	 </div>
 </body>
</html>
```
3. Create a page to display the results of the HubSpot contact search API (i.e. searchresults.ejs):
```
<html>
 <body>
  <h1>Search for contacts in HubSpot</h1>
	 <div class="contactsearch">
	  <form action="/admin" method="post">
		 <input type="text" name="searchinput">
		 <input type="submit" name="search" value="Search for contact">
		</form>
	 </div>
   <div>
	  <% if (contactsdata!='') { %>
		 <table class="formresults" align="center">
		  <tr>
			 <th>Vid</th>
			 <th>First Name</th>
			 <th>Email</th>
			 <th>Lifecycle Stage</th>
			 <th>HubSpot Owner</th>
			 <th>Contact URL</th>
			</tr>
      
			<% for (var i=0; i<contactsdata.length; i++) { %>
			<tr>
			 <td>
			  <% if (typeof contactsdata[i].vid!='undefined') { %>
			   <%= contactsdata[i].vid %>
		          <% } %>
			 </td>
			 <td>
			  <% if (typeof contactsdata[i].properties.firstname!='undefined') { %>
			   <%= contactsdata[i].properties.firstname.value %>
			  <% } %>
			 </td>
			 <td>
			  <% if (typeof contactsdata[i].properties.email!='undefined') { %>
			   <%= contactsdata[i].properties.email.value %>
			  <% } %>
			 </td>
			 <td>
			  <% if (typeof contactsdata[i].properties.lifecyclestage!='undefined') { %>
			   <%= contactsdata[i].properties.lifecyclestage.value %>
			  <% } %>
			 </td>
			 <td>
			  <% if (typeof contactsdata[i].properties.hubspot_owner_id!='undefined') { %>
			   <%= contactsdata[i].properties.hubspot_owner_id.value %>
			  <% } %>
			 </td>
			 <td>
			  <% if (typeof contactsdata[i]["profile-url"]!='undefined') { %>
			   <a target="_blank" href="<%= contactsdata[i]["profile-url"] %>">Link to contact</a>
			  <% } %>
			 </td>
			</tr>
		       <% } %>
		</table>
	<% } else { %>
	 <p>No results</p>
	<% } %>
   </div>
 </body>
</html>
```

### Create get route to display adminInstall.ejs page
Create a get route to display a page that allows visitor to start the OAuth installation flow:
```
app.get('/admin', (req, res) => { 					  	
 if (isAuthorized(req.sessionID)) {
  res.render('admin');
 } else {
  res.render('adminInstall');
 }
});
```

### Create post route for HubSpot contact search API
1. Create a new route handler:
```
app.post('/admin', async (req, res) => {
});
```
2. Within the route handler, add a conditional statement to check if the visitor has went through the OAuth flow to obtain the tokens:
```
 if (isAuthorized(req.sessionID)) {
 } else {
 }
```
3. If the visitor has an access token, allow visitor to perform the contacts search API request:
```
var searchInput = req.body.searchinput; // Store submitted form input into variable 
var url = 'https://api.hubapi.com/contacts/v1/search/query?q=' + searchInput;

const contactSearch = async (accessToken) => {
 try {
  const headers = {
	 Authorization: `Bearer ${accessToken}`,
	 'Content-Type': 'application/json'
	};
	const data = await request.get(url, {headers: headers, json: true});
	return data;
 } catch (e) {
  return {msg: e.message}
 }};

const accessToken = await getAccessToken(req.sessionID);
const searchResults = await contactSearch(accessToken);
var contactResults = JSON.stringify(searchResults.contacts);
var parsedResults = JSON.parse(contactResults);

res.render('searchresults', {contactsdata: parsedResults});
						
```
4. If the visitor does not have an access token, redirect them to the page where they can install the app:
```
res.redirect('/admin');
```

### Test out the OAuth flow and contact search API
1. Access `http://localhost:3000/admin` and click on Install (/install route) to start the OAuth installation flow
2. After completing OAuth flow, you should be redirected to /admin page where it should display the search input bar
3. Try making a search by submitting a query in the search input bar, the page should render searchresults.ejs template and display the search results in a table

### Push changes to Cyclic 
1. Before pushing changes to Cyclic, change the redirect_uri variable to your Cyclic app's domain:
```
// const REDIRECT_URI = `http://localhost:${port}/oauth-callback`;
const REDIRECT_URI = `https://tan-magnificent-slug.cyclic.app/oauth-callback`;
```
2. Commit all of the changes to your local git repository using `git add .` and `git commit -m "update"`
3. Push changes to Heroku using `git push heroku master`
