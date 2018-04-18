# admin-functions

Create, delete and list all users on client side without back-end server

### Using firestore trigger

* #### Create user

```javascript
exports.createUser = functions.firestore
    .document('User/{user}')
    .onCreate((snap) => {
        //you can add any other property present in
        //https://firebase.google.com/docs/reference/js/firebase.User#properties
        let dbvalues = snap.data();
        let name = dbvalues.name;
        let pass = dbvalues.pass;
        let email = dbvalues.email;
        //create user using data of recently add user on firestore
        admin.auth().createUser({
                email: email,
                password: pass,
                displayName: name
            })
            .then(userRecord => {
                //Get the uid of the user newly created and adds to the document
                //referring to the user, this step is very important so that the
                //trigger used to delete the user works
                snap.data.ref.set({
                    uid: userRecord.uid
                }, {
                    merge: true
                });
                //delete user pass field
                snap.data.ref.update({
                    pass: FieldValue.delete()
                });
                console.log('Successfully created new user: ', userRecord);
            })
            .catch((error) => {
                console.log('There was an error creating the user: ', error);
            });
    });
```
* #### Delete user
```javascript
exports.deleteUser = functions.firestore
    .document('User/{user}')
    .onDelete((snap) => {
        //get delete document data
        let dbvalues = snap.data();
        let uid = dbvalues.uid;
        //revoke login token for desconect a conected user
        admin.auth().revokeRefreshTokens(uid)
            .then(() => {
                //delete user using uid present on document
                admin.auth().deleteUser(uid)
                    .then(() => {
                        console.log('Deleted user successfully: ', uid);
                    })
                    .catch(error => {
                        console.log('There was an error deleting the user: ', error);
                    });
            });
    });
```
### Using https request

* #### Create user
```javascript
exports.createUser = functions.https.onRequest((req, res) => {
    cors(req, res, () => {
        //get user token to verify that the user is actually logged in
        const userToken = req.get('userToken');
        //verify the token and returns DecodedIdToken as decoded, containing the
        //list properties here
        //https://firebase.google.com/docs/reference/admin/node/admin.auth.DecodedIdToken
        return admin.auth().verifyIdToken(userToken)
            .then((decoded) => {
                //create user using data received by a post request, you can see
                //more on http://expressjs.com/en/4x/api.html#req.body
                admin.auth().createUser({
                        //you can add any other property present in
                        //https://firebase.google.com/docs/reference/js/firebase.User#properties
                        email: req.body.email,
                        password: req.body.pass,
                        displayName: req.body.name,
                    })
                    .then((userRecord) => {
                        console.log('Successfully created new user: ', userRecord.uid);
                        //send a message to client web if the user is
                        //successfully created
                        res.status(200).send({
                            code: 1,
                            message: 'Successfully created new user',
                            userRecord: userRecord
                        });
                    })
                    .catch((error) => {
                        console.log('Error creating new user: ', error);
                        //send a message to client web if occurred an error on
                        //user creation
                        res.status(400).send({
                            code: 2,
                            message: 'Erro ao criar usuario',
                            error: error
                        });
                    });
            }) //send a message to client web if the user token is invalid
            .catch((err) => res.status(401).send(err));
    });
});
```
* #### Delete user
```javascript
exports.deleteUser = functions.https.onRequest((req, res) => {
    cors(req, res, () => {
        //get user token to verify that the user is actually logged in
        const userToken = req.get('userToken');
        //verify the token and returns DecodedIdToken as decoded, containing the
        //list properties here
        //https://firebase.google.com/docs/reference/admin/node/admin.auth.DecodedIdToken
        return admin.auth().verifyIdToken(userToken)
            .then((decoded) => {
                let uid = req.body.uid;
                //revoke login token for desconect a conected user
                admin.auth().revokeRefreshTokens(uid)
                    .then(() => {
                        //delete user using uid received by a post request and
                        //attributed to "let uid = req.body.uid", you can see
                        //more on http://expressjs.com/en/4x/api.html#req.body
                        admin.auth().deleteUser(uid)
                            .then(() => {
                                console.log('Deleted user successfully');
                                //send a message to client web if the user is
                                //successfully deleted
                                res.status(200).send({
                                    code: 1,
                                    message: 'Deleted user successfully'
                                });
                            })
                            .catch((error) => {
                                console.log('There was an error deleting the user ', error);
                                //send a message to client web if occurred an
                                //error on user delete
                                res.status(400).send({
                                    code: 2,
                                    message: 'There was an error deleting the user',
                                    error: error
                                });
                            });
                    });

            }) //send a message to client web if the user token is invalid
            .catch((err) => res.status(401).send(err));
    });
});
```
* #### List all users
```javascript
exports.listUsers = functions.https.onRequest((req, res) => {
    cors(req, res, () => {
        //get user token to verify that the user is actually logged in
        const userToken = req.get('userToken');
        //verify the token and returns DecodedIdToken as decoded, containing the
        //list properties here
        //https://firebase.google.com/docs/reference/admin/node/admin.auth.DecodedIdToken
        return admin.auth().verifyIdToken(userToken)
            .then((decoded) => {
                //list all users and return listUsersResult, you can see
                //complete properties here
                //https://firebase.google.com/docs/reference/admin/node/admin.auth.ListUsersResult
                admin.auth().listUsers().then((listUsersResult) => {
                    //in this part we assign the "users" property of the
                    //listUsersResult to the variable users, you can see more of
                    //users properties here
                    //https://firebase.google.com/docs/reference/admin/node/admin.auth.UserRecord
                    let users = listUsersResult.users;
                    let objUsers = [];
                    //create an array of objects with information you want to
                    //return
                    for (let i = 0; i < users.length; i++) {
                        //here we exclude the admin user from being listed
                        if (users[i]['email'] !== 'admin@gmail.com') {
                            objUsers[i] = {
                                displayName: users[i]['displayName'],
                                uid: users[i]['uid'],
                                email: users[i]['email']
                            };
                        }
                    }
                    console.log(objUsers.filter(Boolean));
                    //as i had used an "if" to exclude the admin user from the
                    //listing, the "objUser" array would return with one of its
                    //positions as null, to solve this we used the filter
                    //function
                    res.status(200).send(objUsers.filter(Boolean));
                });
            }) //send a message to client web if the user token is invalid
            .catch((err) => res.status(401).send(err));
    });
});
```
