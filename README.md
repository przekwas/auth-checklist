# Protecting Our Server

Authenticating, the act of verifying our user's credentials, and Authorizing, the act of allowing our user to make protected requests, have a million different approaches that vary from app to app. We're learning the basic "workflow" of these two processes and will achieve it with little actual code. The code itself is not important to _memorize_ but it is important to be able to describe what it's doing, and where you need to use it, and why you use it there.

### Installing Node Packages

```bash
npm i passport @types/passport
```

[Passport](http://www.passportjs.org/) is an express middleware that can help streamline our "auth" logic into some easy to use functions. It will handle sending status codes like 401 (Unauthorized) for us automatically, it can extract email and passwords from requests, and tokens too! It saves us the work of writing all these crazy if/else statements in our routes and gives us something easy to add as middleware to any route we want. It has many different strategies! We're gonna focus on two ..
<br />

```bash
npm i passport-local @types/passport-local
```

[Passport Local Strategy](http://www.passportjs.org/packages/passport-local/) this is a middleware for our Passport! Yes .. more middlewares. It's middlewares all the way down! It defines a strategy to authenticate our user with username and password. But we can choose to override username to email, which we will! The workflow of this strategy is simple. Start with a check if the email someone is logging in with exists in our database. If it does, then check to see if their password they are logging in with matches what they have in our database. If everything checks out, great! We're logged in. If the email or password are wrong? BOO 401 Unauthorized try again!
<br />

```bash
npm i passport-jwt @types/passport-jwt
```

[Passport JWT](http://www.passportjs.org/packages/passport-jwt/) is nifty in that it combines several other strategies into this one nifty package. It is meant to find/extract and verify a JWT (JSON Web Token) to make sure it's legit. If so? Then this user is allowed to do stuff! Like posting a blog, editing their old blog, or deleting embarrassing ones. But if the token is no good or not there? BOO 401 Unauthorized try again! This is what will "protect" our various endpoints.
<br />

```bash
npm i jsonwebtoken @types/jsonwebtoken
```

[JSON Web Tokens](https://www.jsonwebtoken.io/) are a newer and popular way of telling our apps if a user is logged in, and confirming it with the server. They are **not** encrypted, but rather encoded. They aren't meant to be secure. Anyone can open one up and look at it. So do not store anything like a password in these guys. Just general info!!! Think of these as room cards at a hotel. It's issued and assigned to us, it can expire after some time, and it can only open your room. Not everyone's.

They have 3 parts, the Header, the Payload, and the Footer. They look like this:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImp0aSI6ImZmYWVjNTE4LTFlZjgtNGViZS04NTExLTVkNjRkOWIwY2Q5NSIsImlhdCI6MTU5ODAzMjQ5MywiZXhwIjoxNTk4MDM2MDkzfQ.LLmYQ8u9AqfhklNQy0aGGIRCehjDbg8vCaJXJ3gm5YI
```

Wow that's crazy right? Header and Footer are auto generated for you and typically just contain meta-data and algorithm stuff. The Payload is what you make it. Our payload in this lecture will be small and easy. Simple who the token belongs to, aka a `userid` property. So our payload will be as simple as:

```js
{
	userid: 1;
}
```

We will present this token to our server with our requests and the server will confirm this token is not tampered with or expired. Passport JWT is what will do this when it gets a JWT.
<br />

```bash
npm i bcrypt @types/bcrypt
```

[Bcrypt](https://www.npmjs.com/package/bcrypt) is a silicon valley standard encryption library. It is immensely popular and easy to use. It does all the hard work for you! It will be responsible for taking a person's password like `percyisakitty2020` and outputting `$2b$12$gSgZrFBv/XrMuXMr7go5O.nZv8Xti5vL2bNijTGKSKBCNybIMtuLO` which is the salt and hashed version of that password! We do **not** want to store plain-text passwords in the database. So we will store the encrypted version instead.

It will also handle comparing those two passwords during a login to make sure the login attempt's password can be matched to that secure hash/salt we have in our database.

Push to your repo after installing these and let me know so I can inspect your `package.json` to confirm this step is done!

### Add To Your Config File

We'll need a "secret" phrase for our tokens coming up, so add to your config object something like this and use whatever phrase you want:

```js
export default {
	mysql: {
		...
	}
	auth: {
		secret: 'Lord-Percival-Fredrickstein-von-Musel-Klossowski-de-Rolo-III'
	}
}
```

Think of this as our personal signature on each token. This will prevent someone from making fake tokens as they will never be able to guess our personal signature. Since this file is in your `.gitignore` you can't push it to github and show me, so send me a DM with your config object pasted in so I can confirm this step is done.

### Queries

We'll need at least two queries for your authors table. So if you don't have an `authors.ts` file in your db folder, make one! We will eventually need to: find a user by their email (like they're trying to log in!) and by their id (classic CRUD stuff). We can actually do this with just 1 function, let's see how:

```js
import { Connection } from './index';

// the ?? is an escape placeholder for column names
// the ? is the same escape character you've used for values!
// Examples:
// await db.authors.find('email', 'luke@covalence.io');
// await db.authors.find('id', 1);
// it works with any column you want from your table :)
const find = (column: string, value: string | number) =>
	Connection('SELECT * FROM authors WHERE ?? = ?', [column, value]);

export default {
	find
};
```

And that's it! Push changes to github and DM me.

### Utilities

We know that we can code everything in one big ol' file but we also know that's not a good idea. We wanna keep things small, compartmentalized, and easy to find/debug in a file with like 20 lines rather than a file with 20,000 lines! So the structure of these is my opinion on a good practice, so let's go!

```js
/* FILE PATH TO MAKE */
// /server/utils/passwords.ts

// import the bcrypt library so we can use it
import * as bcrypt from 'bcrypt';

// a function that will take a plain-text password like percyisakitten2020
// and output a hash like $2b$12$gSgZrFBv/XrMuXMr7go5O.nZv8Xti5vL2bNijTGKSKBCNybIMtuLO
export const generateHash = (password: string) => {
	// the number of salt rounds for encyption, 10-12 is a good baseline
	// https://en.wikipedia.org/wiki/Salt_(cryptography)
	const salt = bcrypt.genSaltSync(12);
	// take our plain-text password, our salt, and generate the hash
	const hash = bcrypt.hashSync(password, salt);
	// return it, we're gonna use it when we "register" a new user!
	return hash;
};

// a function that will take a user's password they're attempting to log in with
// and a hash they have stored in the database
// it will simply return true or false for us!
export const comparePasswords = (password: string, hash: string) => {
	return bcrypt.compareSync(password, hash);
};
```

```js
/* FILE PATH TO MAKE */
// /server/utils/tokens.ts

// import the jsonwebtoken library so we can use it to make a jwt
import * as jwt from 'jsonwebtoken';
// this will be needed for our "secret" signature for the creating a token :)
import config from '../config';

// this will describe what our "payload" that we encode will be, simply who this token belongs to
export interface IPayload {
	userid: number;
}

// this function will make our token!  It takes a payload as the argument
export const createToken = (payload: IPayload) => {
	// jwt.sign makes a token, signed with a secret signature
	// payload is well .. an object with just userid in it
	// config.auth.secret is any string you want!  As long as it is consistent and HIDDEN FROM GITHUB
	// the expiresIn is optional but we don't want a token existing forever, so we choose to expire it in 15 days
	const token = jwt.sign(payload, config.auth.secret, { expiresIn: '15d' });
	return token;
};
```

Add these files and code to your repo and push. DM me when you do!

### Passport

Add a new folder in your server called middlewares and a new file called `passport-strategies.ts`!  
It should look like `/src/server/middlewares/passport-strategies.ts` when you get 'er done. This file will handle our two strategies for passport so it can do it's job.

```js
import * as passport from 'passport';
import * as jwtStrategy from 'passport-jwt';
import * as PassportLocal from 'passport-local';
import db from '../db';
import config from '../config';
import { comparePasswords } from '../utils/passwords';
import type { IPayload } from '../utils/tokens';

// tbh just memorize that these are needed.  Don't ask why lmao.
// its point is to create something called req.user for us to use later
passport.serializeUser((user, done) => done(null, user));
passport.deserializeUser((user, done) => done(null, user));

// Local Strategy is how we have our user fill out a login form on our app, and how we check their credentials in our own database.  It defaults to username and we override it to use email instead
passport.use(
	new PassportLocal.Strategy({ usernameField: 'email' }, async (email, password, done) => {
		try {
			// passport-local gives us the email and password someone is trying to login with
			// so first thing make sure their email even exists in our db, find it!
			const [author] = await db.authors.find('email', email);
			// then we make sure they are real and that their password they're tring to login with matches what we store in our database
			if (author && comparePasswords(password, author.password)) {
				// this is a good security step.  DO NOT pass around a user's password anymore than where you need it
				delete author.password;
				// done(); is the magic function passport uses to know this strategy is well .. done running!
				done(null, author);
			} else {
				// this represents a bad login attempt, and passport will auto send a response for you of 401 Unauthorized, how cool is that!
				done(null, false);
			}
		} catch (error) {
			// classic we effed up lol
			console.log(error);
			done(error);
		}
	})
);
```
