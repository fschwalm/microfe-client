# *microfe*
*(microfe - short for micro frontends)*

A naive infrastructure/meta-framework implementation for micro frontends. This project intends to provide the necessary tooling to achieve independent apps loaded separately and run on different parts on a single web page in complete isolation.
For detailed information on the topic can be found [micro-frontends.org](https://micro-frontends.org/)
## Motivation
When developing microservices there are lots of tools and libraries to help developers to focus the effort on the things needs to be done instead of fighting against a monolithic monster. For now, "micro frontends" idea is still premature and it needs time to grow something easy to use. My intention is to contribute to this discussion and also provide necessary tooling and a sample architecture for developers who would like to give it a try. Providing an easy to use infrastructure for individuals and companies can be considered as an ultimate goal.
## Who will/may/can use *microfe*?
Ideally *microfe* is not suitable for small teams and for them trying to use it would not be necessary. For this kind of teams refactoring their monolithic fe apps would be more productive instead of using *microfe* to divide a relatively big app into smaller pieces and trying to maintain each piece. If the project contains at least two independent teams which are responsible for the same monolithic app then *microfe* can be beneficial. Because *microfe* gives the opportunity of working on independent tech stack by each team. It can provide isolation and managed communication channels between micro-apps.
## On Micro Frontends
While companies growing they usually move from one team to two or more and they start to divide the code base and on the backend side microservice architecture has lots of benefits to scale the company up. On the frontend side, the code base becomes a growing monolith even if it is written in a modular fashion. So scaling a front end team is not so easy and problems start to appear. Lack of communication between teams, conflicting merges, hard to change tech stacks, hard to update dependencies and the list goes on.

Similar to microservices, the micro frontends provides the opportunity to isolate code bases and make the teams free to use any code standards and tech stack and focus on relatively small parts of the application.
## Goals
* Isolated and Independent apps
* A way to have a unified UI
* Inter-app communication (i.e. authentication)
* Easy to maintain apps
* Not to break already available build environments for major frameworks (React, Angular, Vue)
* Freedom of tech stack choice

## Requirements
To run the microfe locally you need to clone and run [micro-fe-registry](https://github.com/onerzafer/micro-fe-registry). The documentation for micro-fe-registry can be found under its own repository.

## Usage
Currently, there is no npm package provided and the usage is not recommended at this phase. Yet if you are willing to experiment by yourself clone both repositories. For micro-fe-registry part follow the instructions on its own repository. Then execute following commands
```bash
npm install
npm start
```
This command will open your browser on http://localhost:8080 and you will be able to see the page is running.

If you see just blank page be sure your micro-fe-registry installation is up and running. And if it is running already please check if the requested micro-apps are available on the registry folder with requested names. If you have still problems of running please open an issue I will be happy to help you.

## Top level architecture
The microfe library basically has 4 different main parts *AppsManager, Loader, Router* and *Store*. It also provides some helper functions and classes: *Bootstrapper, Microfe decorator* and  *provider*.
These all parts of the microfe library can function with a specific micro-apps wich implements microfe interface.

### Definition of a microfe app
A microfe app should implement the following interface
```TypeScript
interface MicroApp {
    name: string;
    deps?: string[];
    initialize: (args: {
        AppsManager: AppsManager;
        Config: ConfigInterface;
        [key: string]: any;
    }) => any | void;
}
```
**initialize** function may return the instance of the app

### Bootstrapper
The responsibility of Bootstrapper is initializing the AppsManager and all other micro-apps provided inside the library. So it can be considered as the entry point of the microfe library. The signature for bootstrapper can be described like this:
```TypeScript
const bootstrap = (
    routes: Route[],
    config: ConfigInterface
) => (...microApps: MicroApp[]) => void
```
To boostrap the microfe meta-framework the following example can be used as a refrence:
```TypeScript
import { Microfe, Bootstrap, Route, ConfigInterface } from './lib';

@Microfe({
    deps: ['LayoutApp'],
})
class Main {
    constructor() {
        console.log('Initialised');
    }
}

const Routes: Route[] = [
    { path: '/', redirectTo: '/angular' },
    { path: '/angular', microApp: 'demoAngular', tagName: 'demo-angular' },
    { path: '/react', microApp: 'reactDemo', tagName: 'react-demo' },
    { path: '/static', microApp: 'staticApp', tagName: 'static-app' },
    { path: '*', microApp: 'NotFoundApp', tagName: 'not-found-app' },
];

const Config: ConfigInterface = {
    registryApi: 'http://localhost:3000/registry',
    registryPublic: 'http://localhost:3000',
};

Bootstrap(Routes, Config)(Main);
```

### AppsManager
The main functionality of AppsManager is creating the dependency tree and when all of the dependencies of a micro-app are ready, it instantiates the micro-app by providing the dependencies instances. The public API for AppsManager can be summarized as follows:
```TypeScript
interface AppsManager {
    register: (app: MicroApp) => void;
    subscribe: (fn: (notFoundApps: MicroApp[]) => void) => {
        unsubscribe: () => void
    }
}
```
AppsManager passes the config and itself as default dependency to all of the instances of provided micro-apps. Which means all have the access to AppsManager and its public API. Alternatively, AppsManager can be accessed from window global as AppsManager.

*AppsManager is the only part which does not implement the MicroApp interface. The rest of the library actually is a collection of micro-apps.*
### Loader
When registered by Bootstrapper the Loader requires Config and waits until it is provided. With the config Loader receives the micro-fe-registry public URLs. After getting the Config AppsManagers instantiates the Loader. On constructer Loader subscribes to AppsManagers and start the not found micro-apps. When a new not found micro-app available the Loader parse the micro-app URL by combining the name of the micro-app and public URL of micro-fe-registry injects it to the dom as a remote script. Naturally, the browser loads the micro-app from given URL. The loader can be a dependency and it has only one public function.
```TypeScript
const Loader {
    fetchMicroApp: (name: string) => void
}
```
### Router
Unlike the common routers, the microfe router has limited functionality. It is capable of solving the first part of the declared URLs. This implementation assumes the rest of the URL will be resolved by the responsible micro-app. If the Router can resolve the URL from the browser location it triggers the Loader.fetchMicroApp function with the name of resolved micro-app. So it has two dependencies routes and Loader.
#### Route
The routes object is an array of the Route objects which has the following interface:
```TypeScript
interface Route {
    path: string;
    tagName?: string;
    redirectTo?: string;
    microApp?: string;
}
```
#### micro-router the Router outlet
When Router instantiates it register a web component called micro-router. This is the expected place for all other micro-apps loads on route hit. The usage is pretty simple and available for all micro-apps living on the client at the moment.
```html
<micro-router></micro-router>
```
Currently, it has no targetting of sub-routes. Which means all of the micro-router tags will display the same target micro-app. So current recommendations are using only one micro-router the page. In the future, some sub-routes can be targetted to some named micro-router tags.
#### micro-link
Router also provides a simple navigation element with no design. All micro-apps will be able to access it any time since it is provided as a web component like micro-router. micro-link has one attribute which is href and if the given path is the current route, it assigns itself automatically active class. So no need to observe history and match correct path to put active class to the links.
```html
<micro-link href="/some/cool/page">
    Some Cool Page
</micro-link>
```
When we navigate to /some/cool/page this micro-link above will be marked as active. 
### Store
With the assumption of only big teams and big code bases will need the microfe and nearly all of the already managing app state, the microfe library provides a global shared inter-app state. this state can be used as a shared event bus or shared global state. By nature, this store is reactive and powered by RxJS. Yet it still has the similar functionalities of Redux library.
```TypeScript
interface Action {
    type: string;
    [key: string]: any;
}

interface State {
    [key: string]: any;
}

interface Reducer {
    (action: Action, state: State) => State) => void;
}

interface ReducerTreePiece {
    [key: string]: Reducer | ReducerTreePiece
}

interface MicroAppStore {
    addReducer: (reducerTreePiece: ReducerTreePiece) => void;
    dispatch: (action: Action) => void;
    select: (selector: string) => Observable<State>;
}
```
The main issue with the MicroAppStore is the reducers may arrive on different times. The select function is pretty useful on this case. Because if the selected reducer is not available it sibly emits undefined and when the reducer arrives it emmits to all subscribers the related state.
```TypeScript
const Todos$ = MicroAppsStore.select('todos');
Todos$.subscribe(todos => console.log(todos)); // immediatelly logs undefined
const todosReducer = (state = [], action: Action) => {
    return state;
}
MicroAppsStore.addReducer({todos: todoRecucer});
// At this point Todos$.subscribe will receive [] as todos and will log []
```
### Microfe decorator
This decorator can be used as an helper for casting any class to a micro-app
```TypeScript
@Microfe({
    deps: ['LayoutApp']
})
export class Main {
    private layoutApp;
    constructor({LayoutApp}) {
        this.layoutApp = LayoutApp;
        this.render();
    }

    private render() {
        this.layoutApp.someLayoutAppFunction();
    }
}
```
The code block above will be equavalent to following code:
```javascript
{
    name: 'Main',
    deps: ['LayoutApp']
    initialize: function({LayoutApp}) {
        LayoutApp.someLayoutAppFunction();
    }
}
```
### Provider
The provider is a helper function to provide objects as micro-app. So any static data can be provided to other micro-apps with Provide function:
```TypeScript
const languageEn = {hello: 'Hello'};
const languageEnProvider = provide(languageEn);
```
Then languageEnProvider can be passed down to all micro-apps which has the dependency as follows:
```TypeScript
Bootstrap(Routes, Config)(Main, LanguageEnProvider);
```
# License
[MIT](https://choosealicense.com/licenses/mit/)