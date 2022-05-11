# Testing business actions

## Example social network app

In my application, in the rest server API, I want to process the requests coming from the frontend. Typical web-based application will have a client in javascript performing requests to the server via HTTP requests.

Let's consider an application that will an social network application. This application will have a feature to register a new user. If I want to use this application, I want to register a profile, so I can interact with other user.

On a technical side, in the user registration page, this application would have a client code, that is calling REST endpoint with new user data and the REST endpoint would process the data and anwser with success/failure code.

Client code will gather user input and post the data to the server:
```
type UserData = {
    email: string,
    nickname: string
}
const userData: UserData = {
    email: "user123@gmail.com",
    nickname: "user_123"
}
const response = httpRequester.post(`/api/user`, userData)
...
```

On server-side, the server would handle the request:

Firstly, there would be a handler to dispatch the HTTP request to correct part of code, that is dealing with user registrations. Let's call it handler
```
app.post(`/api/user`, (request, response) => {
    const userData = request.body as UserData
    // ... validation ...
    saveUserData(userData)
})
```

Secondly, there would be a code that deals with code saving the data to the database. Let's call it *action*:

```
function saveUserData(userData: UserData) {
    const userId = dataSource.save(User, userData.email)
    if (userId) {
        dataSource.save(UserProfile, userId, userData.nickname)
        return "success";
    } else {
        return "failure";
    }
}
```

## Integration testing of code

Above client & server code, albeit contrived, represents typical client server interaction of an app. This coud would work, but there is a room for improvement. When I want to test this code, I can only do some end-to-end test, I need to start the backend test server, startup the database, then create a http request, and check if the data in database is correct. But having these kind of test is not cheap, these kind of test require a lot of maintenance. [More on this on google blog](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)

Instead of that, I might like to have an integration test, that would only test the saveUserData *action* without touching any database and handler code. In that way, I can check, if my business logic inside the `saveUserData` *action* is correct without dealing with details, that are not relevant to that.

First thing that I need to do, is to deal with the fact, that `dataSource` is could bound to database. Since I want to avoid dealing with database, I don't want to depend on real database in the code of *action*, but I want to use inversion of control technique and inject the code dealing with storing/retrieving users as a parametr to saveUserData function.

So, I will rewrite the `saveUserData` *action* as follows:
```
function saveUserData(userData: UserData, userStore: UserStore) {
    const userId = userStore.saveUser(userData.email)
    if (userId) {
        userStore.saveUserProfile(userData.nickname)
        return "success";
    } else {
        return "failure";
    }
}
```

User service will be a collection of functions dealing with storing and retreiving user from the database:

```
interface UserStore {
    saveUser: (email: string) => number | null;
    saveUserProfile: (userId: number, nickname: string) => void;
}
```

Then, I change the behaviour of `saveUserData` function by providing with different implementations of `UserStore`. While in production code, the implementation of the `UserStore` fields would be as in `saveUserData` function before it got the `UserStore` parameter, the implementation of `UserStore` would end up looking like this: 
```
const userStoreReturningNull: {
    saveUser: () => (null),
    ...
}
```
Whet the `UserStore` is implemented like this, I can check in test code, how the `saveUserData` *action* behaves when the saving of user data fails. 
```
test("When saving of user fails, the profile is not saved") {
    const result = saveUserData(testData, testStore)
    assert(testStore.saveUserProfile).notCalled()
}
```

This is an example of integration test, that will do two things, firstly, assure quality of code in test and serve as documentation.

## Growing of `UserStore`

When more endpoints will be added, one strategy how to deal with this would be to add more methods to `UserStore`. There will be endpoints for deleting user, updating user and other operations. If we would add these methods as fields to `UserStore`, then it would end up looking like this:

```
interface UserStore {
    saveUser: (email: string) => number | null;
    saveUserProfile: (userId: number, nickname: string) => void;
    ...
    deleteUser: (userId: number) => void;
    updateUser: (userId: number, email: string) => void;
    ...
}
```

But having this `UserStore` interface, it is not optimal to require it as dependency in the `saveUserData` *action*. Why? Because in this way, we are violating the **I** of SOLID principle, that is [Interface segregation principle](https://en.wikipedia.org/wiki/Interface_segregation_principle). The `saveUserData` would then depend on methods like `deleteUser`, which it never uses. That would make the intention of the code less clear. One option would be insted to depend on a subset of `UserStore`, that contains only the fields, that are actually used in the `saveUserData` function. This can be implemented in typescript by using it's capabilities:
```
type SaveUserDataStore = Pick<UserStore, "saveUser" | "saveUserProfile">
```
This approach has two caveats:
* some other *action* can require different implementation with the same name, for example `updateUser` in one *action* would require different parameters than in another *action*
* it might not be a good idea to lump all the function in a single interface anyway. That could end up having tens of fields.

Bulletproof approach to this is not to have some `UserStore` interface serving all the *actions* in app dealing with user, but to have a custom set of dependecies, that have to be implemented for each *action*. We can thus rewrite create it like this:
```
interface SaveUserDataStore {
    saveUser: (email: string) => number | null;
    saveUserProfile: (userId: number, nickname: string) => void;
}
interface DeleteUserDataStore {
    deleteUser: (userId: number) => void;
}
```
In this way, every *action* will only depend on functions, that it actually uses, not violating the *Interface segregation principle*. 

## Further refining store

Another issue, that can pop up is that the production code can deal with datatypes, that have fields, that are unused in the business code. Consider this code:

type Functions = {
    getUser: (userId: number) => User;
    saveUser: (user: User) => void;
}

```
function makeUserAdmin(userId: number, functions: Functions) {
    const user = functions.getUser(userId);
    const updatedUser = {
        ...user,
        admin: true
    };
    functions.saveUser(user)
}
```

One can note, that whatever the `User` type is, only `admin` field is actually accessed in the `User` type. In production, `User` implementation would typically have much more fields, like `id`, `updated_at`, `created_at` and many others that actually are not involved in the business logic inside the `makeUserAdmin` *action*.

With that logic, we can change the definition of the `User` 
```
type ThisUser = { admin: boolean }
```
But this won't work, since the `User` in saveUser requires all the fields from `User`, not just `admin`. In test code, this would suffice, but in production, it wouldn't. In production, this code needs to contains all the fields in production `User`. The correct type is thus:
```
type Functions<ThisUser extends { admin: boolean }> = {
    getUser: (userId: number) => ThisUser;
    saveUser: (user: ThisUser) => void;
}
```

This signature will then accomplish two goals of the design:
- make the business code not depend on anything more above what is needed in the business logic
- being able to call this function in test & production context

## Avoiding boilerplate

Common pattern would be, before calling some *action* that has `functions` parameters representing the injected dependencies, to construct the `functions` objects. How do I know the type of this `functions` object? One way is to extract the functions into Interface. That is quite straightforward when the intrerface doesn't involve any type parameters. But when it does, the type parameters needs to be passed into the interface, adding more boilerplate to the code than is actually needed.

Consider this code: 
```
interface Functions<User extends {admin: boolean}> { 
    getUser: (userId: number) => User
}
function makeUserAdmin<User extends {admin: boolean}>(userId: number, functions: Functions<User>) {}
```
I need to pass the `User` type parameter to the interface. If the type is written directly into the `functions`, there is no such need:
```
function makeUserAdmin<User extends {admin: boolean}>(userId: number, functions: {
    getUser: (userId: number) => User
}) {}
```
If I need, I can still access the `functions` type via typescript's `typeof` operator. But currently, it is not possible to call the function with type arguments on type level. Therefore, one needs to have a workaround, to curry the function.
```
function makeUserAdmin<User extends {admin: boolean}>() {
    return (
        userId: number, 
        functions: {
            getUser: (userId: number) => User
        }) => {
    ...
    }
}
const mua = makeUserAdmin()    
type Functions = Parameters<typeof mua>[1]
```

In this way, I can get the default `Functions` with the type, so that the type parameters are just to suffice the condition. In other words, having `User extends {admin: boolean}` then the call, without explicit type parameters will be `{admin: boolean}`.
