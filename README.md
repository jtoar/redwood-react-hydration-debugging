### Reproducing the hydration error in @redwoodjs@latest (React 17)

- clone this repo
- checkout the react-17 branch (`git checkout react-17`)
- `yarn install`
- `git checkout node_modules/@redwoodjs/core/config/webpack.common.js`
  - this adds back the change I made to Redwood's webpack config so that we're using the development build of React (the error doesn't show up in the production build of React)
- `yarn rw build`
- `yarn rw serve`
- go to http://localhost:8910/about
- open the console; you should see...

```
Warning: Did not expect server HTML to contain the text node "
    " in <div>.
```

<img width="1522" alt="image" src="https://user-images.githubusercontent.com/32992335/224127973-58f3b6d9-b676-4a94-9195-bd3f667218d9.png">

### Reproducing the hydration error in @redwoodjs@canary (React 18)

- clone this repo
- checkout the react-18 branch (`git checkout react-18`)
- `yarn install`
- `yarn rw build`
- `yarn rw serve`
- go to http://localhost:8910/about
- open the console; you should see...
