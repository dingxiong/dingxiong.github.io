# About

Follow https://github.com/cotes2020/chirpy-starter to get start.

# Local testing

```
bundle exec jekyll serve
```

# Add yarn and related tools

Install nvm, node and yarn if not already

```
nvm install 20.10.0
echo 20.10.0 > .nvmrc
npm install --global yarn
```

Initialize the repo.

```
yarn init -2
```

Install husky

```
yarn add husky -D
npm pkg set scripts.prepare="husky install
yarn prepare
npx husky add .husky/pre-commit "npm test"
```

Install linter and lint-staged

```
yarn add -D prettier
yarn add -D lint-staged
```

Add a `lint-staged` block and also, add `yarn lint-staged` in `.husky/pre-commit`.

Node, every time, we need do `nvm use` before commit.
