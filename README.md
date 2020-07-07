## Running the sample

The sample is provided with basic integration of Web SDK. To test it, set the appropriate connection configuration in `scripts/settings.js`, and run a local web server. Some popular tools for local testing are [JavaScript HTTP Server](https://www.npmjs.com/package/http-server), and [Python Web Server](https://developer.mozilla.org/en-US/docs/Learn/Common_questions/set_up_a_local_testing_server#Running_a_simple_local_HTTP_server).

You can also directly open the index.html in Firefox or Safari to interact with the chat widget after setting `URI` and `channelId` in `scripts/settings.js`, without needing to run a local web server.


To test the client auth enabled mode, you will need a library to generate JWT tokens. We recommend the tokens be generated at your backend server and set in the file. To do a local testing, you can use libraries like [jsrsasign](https://github.com/kjur/jsrsasign), or [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) to generate tokens. A more comprehensive list of libraries are available at [jwt.io](https://jwt.io/).
