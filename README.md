# react-scripts@4 Issue with Importing CSS from Another CSS from a Sibling Monorepo Package

This repo reproduces the issue facebook/create-react-app#10373.

## Steps to Reproduce

1. Check out the repository
1. Do `yarn && yarn build`
1. Build fails

Error:

```
./src/index.css
ModuleNotFoundError: Module not found: Error: You attempted to import ../../package1/index.css which falls outside of the project src/ directory. Relative imports outside of src/ are not supported.
You can either move it inside src/, or add a symlink to it from project's node_modules/.
```

## Issue Description

The "culprit" of the error is import statement in [index.css](packages/app/src/index.css):

```css
@import '~@my-company/package1/index.css';
```

It imports CSS from a sibling package in the monorepo controlled by `yarn workspaces`. Note that it's not
a relative import, yet a proper import from `node_modules`.

Notes:
 - This issue is a behavior change in `react-scripts@4`, identical example with `react-scripts@3` works.
 - Import is basically identical to JS import in [index.js](packages/app/src/index.js) from the same
   `@my-company/package1`. JS import works fine, so it's reasonable to expect CSS import to be fine as well.

## Repo Structure

It's a monorepo controlled by [yarn workspaces](https://classic.yarnpkg.com/en/docs/workspaces/). There are
two packages:
1. [@my-company/package1](packages/package1/package.json) - simple package from which files are imported
1. [@my-company/app](packages/app/package.json) - application created with `yarn create react-app app`

The application attempts to make imports from the `package1` package.

## Issue Observations / Troubleshooting

The build error originates from [ModuleScopePlugin.js](https://github.com/facebook/create-react-app/blob/master/packages/react-dev-utils/ModuleScopePlugin.js).
With `react-scripts@3` this CSS import falls into
[this `if` condition](https://github.com/facebook/create-react-app/blob/master/packages/react-dev-utils/ModuleScopePlugin.js#L31).
Since monorepo has symlinks from `node_modules` to the packages, either `request.descriptionFileRoot` is not
a resolved symlink and therefore contains `node_modules`, or it's a resolved symlink but then
`request.__innerRequest_request` isn't defined.

With `react-scripts@4` when it hits this condition, `request.descriptionFileRoot` is a resolved symlink,
so it doesn't contain `node_modules`, and `request.__innerRequest_request` does have a value.
