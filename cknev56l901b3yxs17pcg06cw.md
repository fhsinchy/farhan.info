## Rock Solid Express Application Architecture

Without any doubt [Express](https://expressjs.com/) is one of the most popular web frameworks out there. After its initial release on November 16, 2010, Express is still going strong with 50k+ stars on GitHub and being the base for a number of new web frameworks including [Sails.js](https://sailsjs.com/),  [NestJS](https://nestjs.com/) ,  [Feathers](https://feathersjs.com/)  being the most popular ones.

Part of Express's charm is its simplicity. It's fast, unopinionated and minimal. In other words, Express provides you with a powerful middleware system good enough for building great applications and lets you run free. You can make an entire application within a single `app.js` file or a robust monolith by replicating the MVC pattern and Express won't complain. This level of freedom while not exclusive to Express (looking at you  [Koa](https://koajs.com/) ) is a blessing and a curse, because you never know the architecture you're following is good enough or not **in the long run**.

In these situations, the [Express application generator](https://expressjs.com/en/starter/generator.html) although helpful, is outdated and can not keep up with the needs of modern scalable web applications. Apart from this generator, there are a number of popular project boilerplates out there.

One of these boilerplates is [sahat/hackathon-starter](https://github.com/sahat/hackathon-starter) with 30k+ stars on GitHub and a huge set of features to get you started with. But what I dislike about this boilerplate is the fact that it's too bloated for my needs. I mostly build APIs with Express and this boilerplate comes with a hefty view layer. I surely can cut that off but other parts of the boilerplate are also well suited for full-stack applications rather than REST APIs.

The second one and the one I like the most is [santiq/bulletproof-nodejs](https://github.com/santiq/bulletproof-nodejs/) boilerplate with 3k+ stars on GitHub. Unlike the previous one, this boilerplate is much lighter and very well suited for building APIs. It's also written in  [TypeScript](https://www.typescriptlang.org/)  which is a plus and follows a number of design patterns that I like. Although a robust boilerplate I have **two** issues with this. They are as follows -

* I don't do TypeScript that often hence the DI container implementation won't work well for me.
* The code in this boilerplate is structured by technical roles of the files instead of components.

If you want to learn more about this architecture, you may read this [blog](https://softwareontheroad.com/ideal-nodejs-project-structure/) post.

After looking for a boilerplate that suits me and building few APIs myself, I've finally come up with a rock solid architecture that in my opinion strikes the right balance between features, best-practices and simplicity.

%[https://github.com/fhsinchy/express-mongo-api-boilerplate]

In this article I'll discuss the goods, bads and uglies of express application architecture as well as how I came up with this architecture, why I think this is good and how you can use or extend this project for your needs.

## Table of Content

* [Folder Structure](#folder-structure)
* [The `app` and `server` Instances](#the-app-and-server-instances)
* [Components](#components)
* [The Web and Service Layers](#the-web-and-service-layers)
* [Closing Thoughts](#closing-thoughts)

## Folder Structure

The top level folder structure is as follows -

```shell
.
├── docker-compose.yaml
├── LICENSE
├── Makefile
├── README.md
└── src
```

The `Makefile` contains common bash commands that I use for tasks like starting and stopping the containers, seeding data, seeing logs etc. The `docker-compose.yaml` file contains definitions for the different services such as the database service, the API itself and a [mongo-express](https://github.com/mongo-express/mongo-express) service for easy database administration. Source code for the API lives inside `src`. Top level structure of that directory is as follows -

```shell
.
├── app.js
├── bin
├── config
├── Dockerfile
├── Dockerfile.dev
├── Dockerfile.test
├── jest.config.js
├── log
├── nodemon.json
├── package.json
├── routes.js
├── seed.js
└── seeds
```

Directory and file names are pretty self explanatory. The `config` directory holds configuration for the database, cors, seeder, authentication and some common app oriented configuration.

The `Dockerfile`, `Dockerfile.dev` and `Dockerfile.test` files are used for building the production, development and test images respectively.

Daily generated logs are stored inside `log` directory. A log file for each day is generated using `<project-name>-<date>.log` naming pattern.

The `routes.js` file is responsible for registering different routes with the `app` instance.

The `seeds` directory contains database seeds and `seed.js` is a simple seeder script that we'll come back to later on.

The `nodemon.json` and `jest.config.js` files are configuration files for  [nodemon](https://nodemon.io/)  and  [Jest](https://jestjs.io/)  testing framework.

## The `app` and `server` Instances

When I say `app` instance and `server` instance what I'm actually referring to is - 

```javascript
/**
 * app instance initialization.
 */

const app = express();

/**
 * server instance initialization.
 */

const server = http.createServer(app);
```

For further explanation we'll first have to have a look at the `app.js` and `bin/www` files. Content of the `app.js` file is as follows -

```javascript
/**
 * Module dependencies.
 */

const cors = require('cors');
const { join } = require('path');
const logger = require('morgan');
const helmet = require('helmet');
require('pkginfo')(module, 'name');
const express = require('express');
const rfs = require('rotating-file-stream');
const { isCelebrate } = require('celebrate');
const cookieParser = require('cookie-parser');

const config = require('./config');

/**
 * app instance initialization.
 */

const app = express();

/**
 * Middleware registration.
 */

app.use(cors(config.cors));
app.use(helmet());
app.use(express.json());
app.use(cookieParser());

/**
 * Logger setup.
 */

app.use(logger('common'));
app.use(
  logger('combined', {
    stream: rfs.createStream(
      `${module.exports.name}-${new Date()
        .toISOString()
        .replace(/T.*/, '')
        .split('-')
        .reverse()
        .join('-')}.log`,
      {
        interval: '1d',
        path: join(__dirname, 'log'),
      },
    ),
  }),
);

/**
 * Route registration.
 */

require('./routes')(app);

/**
 * 404 handler.
 */

app.use((req, res, next) => {
  const err = new Error('Not Found!');
  err.status = 404;
  next(err);
});

/**
 * Error handler registration.
 */

app.use((err, req, res, next) => {
  const status = isCelebrate(err) ? 400 : err.status || 500;
  const message =
    config.app.env === 'production' && err.status === 500 ? 'Something Went Wrong!' : err.message;

  if (status === 500) console.log(err.stack);

  res.status(status).json({
    status: status >= 500 ? 'error' : 'fail',
    message,
  });
});

module.exports = app;

```

As you can see this file is used for bootstrapping and exporting the Express `app` instance. This app then gets imported inside `bin/www` file. Content of the `bin/www` file is as follows -

```javascript
#!/usr/bin/env node

/**
 * Module dependencies.
 */

const http = require('http');
const mongoose = require('mongoose');

const app = require('../app');
const config = require('../config');

/**
 * Get port from environment and store in Express.
 */

const { host } = config.app;
const { port } = config.app;
app.set('port', port);

/**
 * Create HTTP server.
 */

const server = http.createServer(app);

/**
 * ODM initialization.
 */

mongoose
  .connect(config.db.connectionString, config.db.connectionOptions)

  .catch((err) => console.log(err));

mongoose.connection.on('error', (err) => {

  console.log(err);
});

/**
 * Listen on provided port, on all network interfaces.
 */

console.log(`app running -> ${host}:${port}`);
server.listen(port);
```

This file is responsible for setting up `mongoose` and spinning up the `server` instance. Although this file should be responsible for the later task only, I had to put my `mongoose` initialization code here avoid cyclic depedency issues. I'll surely find a better place to put this code.

Keeping the `app` instance separate from the `server` facilitates testing in isolation. The `app` instance and `server` instances can imported inside test files and tested without bumping into each other.

The route registration here is another story. I like my `app.js` file frozen. Which means I don't want this file to change every now and then. That's why I've moved the route registration logic to another file `routes.js`. Content of the file is as follows -

```javascript
module.exports = (app) => {
    app.get('/', (req, res) => {
        res.status(200).json({
            error: false,
            message: 'Bonjour, mon ami',
        });
    });
}
```

This file exports an arrow function that takes the `app` instance as parameter. Route middleware are then attached to this instance. Call to this exported function can be seen in `app.js` file -

```javascript
// ...

/**
 * Route registration.
 */

require('./routes')(app);

// ...
```

This way I can keep the `app.js` file away from frequent changes and registering routes in a separate file keeps the code cleaner.

## Components

A common pattern seen not only in Express but also in other platforms is to group code by their technical role instead of components. One example can be as follows -

```shell
.
├── app.js
├── bin
├── controllers
├── helpers
├── middleware
├── migrations
├── models
└── tests
```

This is one of my older projects. As you can see, code is grouped according to their technical roles inside `models`, `migrations`, `controllers`, `middleware` and `helpers` directories. Although it works fine for small projects, you'll find these directories extremely cluttered as the project grows. At this moment this code-base holds ~30 files inside each of those directories.

A better approach is to group files according to components. In an e-commerce application, possible components can be as follows -

* auth - handles authentication and authorization
* admin - handles administrative tasks
* cart - people put their stuff here
* shop - deals with indexing and showing the products
* inventory - handles stock management for the admins

As you can imaging the `auth` component for example holds necessary route handlers, middleware and business logic to handle authentication, authorization features. The structure of this component can be as follows -

```shell
.
├── api
├── middleware
├── models
└── services
```

The `api` directory holds necessary logic for handling the http requests. These can be treated as the controllers. The `middleware` directory holds the middleware (duh) such as the `authenticate` middleware responsible for guarding routes from unauthenticated access. Together these two directories make up the web layer.

The `models` directory holds the database models (schemas) and the `services` directory is the service layer for this component.

## The Web and Service Layers

It's  a common practice to divide web applications into three separate layers namely -

* Web Layer
* Data Access Layer
* Service Layer

The previously mentioned `api` directory along with the `middleware` directory inside a component can be treated as the web layer, responsible for transporting requests and responses. The data access layer is usually responsible for working with the database directly. But as we're using `mongoose` instead of working with the database directly, we can omit this layer.

The service layer is in my opinion the most important. It holds necessary business logic for performing various tasks such as registering a user in case of the `auth` component.

To better understand this concept, lets have a look at the content of the `auth/api/routes/auth.js` file -

```javascript
const { Router } = require('express');
const { celebrate, Joi } = require('celebrate');

const { User } = require('../../models');
const config = require('../../../config');
const { AuthService } = require('../../services');
const { authenticate } = require('../../middleware');

const router = Router();

const authService = new AuthService(User);

module.exports = (routes) => {
  routes.use('/auth', router);

  router.post(
    '/register',
    celebrate({
      body: Joi.object().keys({
        name: Joi.string().trim().required(),
        email: Joi.string().email().trim().required(),
        password: Joi.string().required(),
      }),
    }),
    async (req, res, next) => {
      try {
        res.status(201).json({
          status: 'success',
          message: 'User Registered!',
          data: {
            user: await authService.signup(req.body),
          },
        });
      } catch (err) {
        next(err);
      }
    },
  );

  router.post(
    '/login',
    celebrate({
      body: Joi.object().keys({
        email: Joi.string().email().trim().required(),
        password: Joi.string().required(),
      }),
    }),
    async (req, res, next) => {
      try {
        const { accessToken, refreshToken } = await authService.login(req.body);

        res.cookie('refreshToken', refreshToken, config.auth.refreshToken.cookie.options);

        res.status(200).json({
          status: 'success',
          message: 'User Logged In!',
          accessToken,
        });
      } catch (err) {
        next(err);
      }
    },
  );

  router.post('/logout', authenticate, (req, res) => {
    res.clearCookie('refreshToken');

    res.status(200).json({
      status: 'success',
      message: 'Logged Out!',
    });
  });
};

```

This file is only responsible for transporting requests and responses as I've already mentioned and yes, I consider validating a part of the process. Many people put validation in a separate layer but that seems like overengineering to me. The business logic necessary for performing the requested task is inside the `AuthService` class exported from `services/auth.js` file -

```javascript
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

const config = require('../../config');

module.exports = class AuthService {
  constructor(User) {
    this.User = User;
  }

  async signup(params) {
    if (await this.User.findOne({ email: params.email }).exec()) {
      const err = new Error('Email Already Taken!');
      err.status = 400;
      throw err;
    } else {
      const user = await this.User.create({
        name: params.name,
        email: params.email,
        password: await bcrypt.hash(params.password, 12),
      });

      return {
        name: user.name,
        email: user.email,
      };
    }
  }

  async login(params) {
    const user = await this.User.findOne({ email: params.email }).exec();

    if (!user) {
      const err = new Error('Wrong Email!');
      err.status = 400;
      throw err;
    } else if (await bcrypt.compare(params.password, user.password)) {
      const tokenPayload = {
        name: user.name,
        email: user.email,
      };

      const accessToken = jwt.sign(tokenPayload, config.auth.accessToken.secret, {
        expiresIn: config.auth.accessToken.validity,
      });

      const refreshToken = jwt.sign(tokenPayload, config.auth.refreshToken.secret, {
        expiresIn: config.auth.refreshToken.validity,
      });

      return {
        accessToken,
        refreshToken,
      };
    } else {
      const err = new Error('Wrong Password!');
      err.status = 400;
      throw err;
    }
  }
};
```

As you can see, the `AuthService` class takes the `User` model as a dependency. The actions inside the service receives the request parameters passed by the route handlers, performs necessary actions and returns the processed output. The web layer then returns the values to the client.

The beauty of this approach is that the business logic can now be called from anywhere since it's not a part of the web layer. All it requires is some dependencies and it's usable even from command line applications.

A good example is the `seed.js` file which is a DIY seeder implementation for `mongoose` utilizing the service layer. I'll write about this in another article.

These services can be tested in isolation as well without hitting the `app` instance.

## Closing Thoughts

The architecture of your application will always be dictated by your necessities. This boilerplate or architecture is by no means perfect. It's something that I've been using in medium to large scale APIs (both REST and GrahphQL) for quite some time and it hasn't let me down even once. Look around the code-base, if you think it's suitable for your use-case, leave a start and use this as template. If you think something can be improved, let me know or heck you can just contribute directly. That's the beauty of open-source isn't it?