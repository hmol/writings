![locks](locks.jpeg?raw=true)

# Building a Node API with token based authentication

When building an api in Node I suddenly got the need to implement some form of simple authentication. I wanted to be able to create users and have them log in and access a bunch of secure routes. The solution turned out to be a simple combination of jwt, passport and bcrypt.

[**JWT**](https://jwt.io/) or JSON Web Token is an open standard for representing claims securely between parties. Each request must include the JWT-token in the http header, allowing the user to access routes permitted with that token.

[**Bcrypt**](https://en.wikipedia.org/wiki/Bcrypt) is a password hashing function that we will use to create a hash of the users password to store in database. It is based on the [Blowfish](https://en.wikipedia.org/wiki/Blowfish_cipher) block cipher algorithm.

[**Passport**](http://www.passportjs.org) is an authentication middleware. It has a [strategy](http://www.passportjs.org/packages/passport-jwt/) to automatically handle JWT tokens that we will make use of.

To start with, we need an endpoint that the user can use to log in. This route should not be protected. If the username and password is correct, a valid JWT token is returned in the response. This token must then go in the Authorization header in all of the consecutive HTTP requests to the protected routes.

The http communication will look something like this:

![http2](http.png?raw=true)

I have a working example in my [repository](https://github.com/hmol/node-api-token-auth) which will be explained in detail below. All code is written in typescript, and for this project I have used [express](https://expressjs.com/) as a web application framework.

First we will need controllers to handle requests. The code for the login function lies in the <code>authController</code>. 


```JavaScript
// authController.ts    
import * as jwt from 'jwt-simple';
import * as bcrypt from "bcryptjs";
import moment from "moment";
import user from '../models/user';
import repository from '../utils/repository';
import passportHelper from '../utils/passportHelper';

class authController {
    // generate valid jwt token
    private getToken = (user: user): Object => {
        let expires = moment().utc().add({ days: 7 }).unix();
        let token = jwt.encode({
            exp: expires,
            userid: user.id
        }, passportHelper.jwtSecret);
        
        return {
            token: "JWT " + token,
            expires: moment.unix(expires).format(),
            userid: user.id
        };
    }

    // if username/password is match, return valid jwt token
    public login = async (req: any, res: any) => {
            let user = await repository.getByUsername(req.body.username);

            if (user === null) {
                res.status(500).json({'message': 'user could not log in'});
                return;
            }

            let isMatch = await bcrypt.compare(req.body.password, user.hashedPassword);
          
            if(isMatch) {
                res.status(200).json(this.getToken(user));
            } else {
                res.status(500).json({'message': 'user could not log in'});
            }
    }
}

export default new authController();
```

Then we must define routes for our api:

```JavaScript
// routes.ts
import userController from '../controllers/userController';
import authController from '../controllers/authController';

export = (app: any) => {
    app.post("/api/login", authController.login);

    app.post("/api/users", userController.create);
    app.delete("/api/users/:id", userController.delete);
    app.get("/api/users/:id", userController.getOne);
    app.put("/api/users/:id", userController.update);

    app.get("/api", (req: Request, res: any) => res.status(200).json({ message: "Hello world" }));


    app.use((req: any, res: any, next: any) => {
        res.status(404).json({ "error": "Endpoint not found" });
        next();
    });
};
```

The <code>userController</code> referenced in the code above is just a simple controller that calls on a repository to do standard CRUD operations. No need to elaborate this further. To do these operations however, the repository uses a User model.

```JavaScript
// user.ts
import * as bcrypt from "bcryptjs";

export default class user {
    id: string = '';
    username: string = '';
    hashedPassword: string = '';
    
    constructor(username: string, hashedPassword: string, id: string = '') {
        this.username = username;
        this.hashedPassword = hashedPassword;
        this.id = id;
    }
    
    // bcrypt to compare stored password hash with password from user input
    public async comparePassword(inputPassword: string): Promise<boolean> {
        let hashedPassword = this.hashedPassword;
        return new Promise((resolve, reject) => {
            bcrypt.compare(inputPassword, hashedPassword, (err, success) => {
                if (err) return reject(err);
                return resolve(success);
            });
        });
    };

    // create instance of User with correct hashed password
    static async create(username: string, password: string, id: string = ''): Promise<user> {
        let hashedPassword = await this.hashPassword(password);
        return new user(username, hashedPassword, id);
    }

    // use bcrypt to create hash of password
    static async hashPassword(password: string): Promise<string> {
        const saltRounds = 11;
      
        const hashedPassword = await new Promise<string>((resolve, reject) => {
          bcrypt.hash(password, saltRounds, function(err, hash) {
            if (err) reject(err);
            resolve(hash);
          });
        });
      
        return hashedPassword;
      };
}
```

To separate some of the code for customizing the passport-strategy, we need a helper class:


```JavaScript
// passportHelper.ts
import Repository from './repository';
import user from '../models/user';
const passport = require("passport");
import { Strategy, ExtractJwt } from 'passport-jwt';

class passportHelper {
    jwtSecret = '^RJ3XFYv542jLL@jjG7Zxa1Ihe%9KmXiUEfOH$3iG8q*0f@J!r';

    init() {
        passport.use("jwt", this.getPassportStrategy());
    }

    authenticate = (callback: any) => passport.authenticate("jwt", { session: false, failWithError: true }, callback);

    // get strategy for passport
    getPassportStrategy(): Strategy {
        const params = {
            secretOrKey: this.jwtSecret,
            jwtFromRequest: ExtractJwt.fromAuthHeader(),
            passReqToCallback: true
        };

        return new Strategy(params, (req: any, payload: any, done: any) => {
            Repository.get(payload.userid)            
                .then((user: user) => {
                    if(user === null) {
                        return done(null, false, { message: "The user in the token was not found" });
                    }
                    return done(null, { _id: user.id, username: user.username });
                })
                .catch((err: any) => {
                    return done(err);
                });
        });
    }
}

export default new passportHelper();
```

The last thing we need now is an <code>server.ts</code> to get <a href="https://expressjs.com/">express</a> up and running.

```JavaScript
// server.ts
import passportHelper from './utils/passportHelper';
const bodyParser = require("body-parser");
const express = require("express");
const passport = require("passport");

// setup express and passport
let app = express();
app.use(bodyParser.json());
app.use(passport.initialize());
passportHelper.init();

// run authenticate() for all routes except /api/login
app.all("/api/*", (req: any, res: any, next: any) => {
    if (req.path.includes("/api/login")) return next();
    return passportHelper.authenticate((err: any, user: any, info: any) => {
        if (err) { return next(err); }
        if (!user) {
            return res.status(401).json({ message: 'Token not valid' });
        }
        app.set("user", user);
        return next();
    })(req, res, next);
});

const routes = require("./utils/routes")(app);

const server = app.listen(3000, () => {
    console.log(`Server started on port 3000.`);
});
```

As defined above the `authenticate()` function will run for all routes except `/api/login`. This function will use passport with the defined passport strategy to extract the JWT token from the header and then check if it is valid. If the token is not valid, the response will be `HTTP 401 Unauthorized`.

### The end

If you are working on a large enterprise application, then you should probably think twice before you create your own solution for authenticating users. Often it could be a good idea to use an external provider for this. Nevertheless, this blog post was an example showing how simple it could be to implement stateless authentication with jwt tokens in node. 




*Further reading:*

All code: https://github.com/hmol/node-api-token-auth/

http://www.passportjs.org

https://jwt.io/

https://auth0.com/blog/hashing-in-action-understanding-bcrypt/
