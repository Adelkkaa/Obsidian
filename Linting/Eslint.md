### Создание eslint плагина

1) `volta install yo`
2) `volta install generator-eslint`
3) `yo eslint:plugin`
4) `yo eslint:rule`
5) Пример заполнения правила, которое проверяет нужны ли здесь относительные пути в FSD вместо абсолютных 
```js
/**
 * @fileoverview path checker
 * @author adelkkaa
 */
"use strict";

const path = require('path');


//------------------------------------------------------------------------------
// Rule Definition
//------------------------------------------------------------------------------

/** @type {import('eslint').Rule.RuleModule} */
module.exports = {
  meta: {
    type: null, // `problem`, `suggestion`, or `layout`
    docs: {
      description: "path checker",
      recommended: false,
      url: null, // URL to the documentation page for this rule
    },
    fixable: null, // Or `code` or `whitespace`
    schema: [], // Add a schema if the rule has options
    messages: {}, // Add messageId and message
  },

  create(context) {
    return {
      ImportDeclaration(node) {
        // example app/entities/Article
        const importTo = node.source.value;

        // example C:\Users\tim\Desktop\javascript\production_project\src\entities\Article
        const fromFilename = context.getFilename();

        if(shouldBeRelative(fromFilename, importTo)) {
          context.report(node, 'В рамках одного слайса все пути должны быть относительными');
        }
      }
    };
  },
};

function isPathRelative(path) {
  return path === '.' || path.startsWith('./') || path.startsWith('../')
}

const layers = {
  'entities': 'entities',
  'features': 'features',
  'shared': 'shared',
  'pages': 'pages',
  'widgets': 'widgets',
}

function shouldBeRelative(from, to) {
  if(isPathRelative(to)) {
    return false;
  }
  // example entities/Article
  const toArray = to.split('/')
  const toLayer = toArray[1]; // entities
  const toSlice = toArray[2]; // Article


  if(!toLayer || !toSlice || !layers[toLayer]) {
    return false;
  }


  const normalizedPath = path.toNamespacedPath(from);
  const projectFrom = normalizedPath.split('src')[1];
  const fromArray = projectFrom.split('/')


  const fromLayer = fromArray[1];
  const fromSlice = fromArray[2];

  if(!fromLayer || !fromSlice || !layers[fromLayer]) {
    return false;
  }

  return fromSlice === toSlice && toLayer === fromLayer;
};


```



### Добавление поддержки alias в плагин по проверке абсолютных и относительных путей

```js
/**
 * @fileoverview path checker
 * @author adelkkaa
 */
"use strict";

const path = require("path");

//------------------------------------------------------------------------------
// Rule Definition
//------------------------------------------------------------------------------

/** @type {import('eslint').Rule.RuleModule} */
module.exports = {
  meta: {
    type: null, // `problem`, `suggestion`, or `layout`
    docs: {
      description: "path checker",
      recommended: false,
      url: null, // URL to the documentation page for this rule
    },
    fixable: null, // Or `code` or `whitespace`
    schema: [
      {
        type: "object",
        properties: {
          alias: {
            type: "string",
          },
        },
      },
    ],
    messages: {}, // Add messageId and message
  },

  create(context) {
    const alias = context.options[0]?.alias || '';

    return {
      ImportDeclaration(node) {
        // example app/entities/Article
        const value = node.source.value
        const importTo = alias ? value.replace(`${alias}/`, '') : value;

        // example C:\Users\tim\Desktop\javascript\production_project\src\entities\Article
        const fromFilename = context.getFilename();

        if (shouldBeRelative(fromFilename, importTo)) {
          context.report(
            node,
            "В рамках одного слайса все пути должны быть относительными"
          );
        }
      },
    };
  },
};

function isPathRelative(path) {
  return path === "." || path.startsWith("./") || path.startsWith("../");
}

const layers = {
  entities: "entities",
  features: "features",
  shared: "shared",
  pages: "pages",
  widgets: "widgets",
};

function shouldBeRelative(from, to) {
  if (isPathRelative(to)) {
    return false;
  }
  // example entities/Article
  const toArray = to.split("/");
  const toLayer = toArray[0]; // entities
  const toSlice = toArray[1]; // Article

  if (!toLayer || !toSlice || !layers[toLayer]) {
    return false;
  }

  const normalizedPath = path.toNamespacedPath(from);
  const projectFrom = normalizedPath.split("src")[1];
  const fromArray = projectFrom.split("/");

  const fromLayer = fromArray[1];
  const fromSlice = fromArray[2];

  if (!fromLayer || !fromSlice || !layers[fromLayer]) {
    return false;
  }

  return fromSlice === toSlice && toLayer === fromLayer;
}

```

1) В schema объявлем факт того, что в качестве properties могут передать alias при настройке плагина
2) Достаем этот alias из нашего контекста, если он есть, то просто его убираем из пути, если его нет, то с путем ничего не делаем
3) Также для проверки накатываем тесты
```js
/**
 * @fileoverview path checker
 * @author adelkkaa
 */
"use strict";

//------------------------------------------------------------------------------
// Requirements
//------------------------------------------------------------------------------

const rule = require("../../../lib/rules/fsd-path-checker"),
  RuleTester = require("eslint").RuleTester;

//------------------------------------------------------------------------------
// Tests
//------------------------------------------------------------------------------

const ruleTester = new RuleTester({
});
ruleTester.run("fsd-path-checker", rule, {
  valid: [
    {
      filename:
        "C:/Users/tim/Desktop/javascript/production_project/src/entities/Article",
      code: "import { addCommentFormActions, addCommentFormReducer } from '../../model/slices/addCommentFormSlice'",
      errors: [],
    },
  ],

  invalid: [
    {
      filename:
        "C:/Users/tim/Desktop/javascript/production_project/src/entities/Article",
      code: "import { addCommentFormActions, addCommentFormReducer } from '@/entities/Article/model/slices/addCommentFormSlice'",
      errors: [
        {
          message: "В рамках одного слайса все пути должны быть относительными",
        },
      ],
      options: [
        {
          alias: "@",
        },
      ],
    },
    {
      filename:
        "C:/Users/tim/Desktop/javascript/production_project/src/entities/Article",
      code: "import { addCommentFormActions, addCommentFormReducer } from 'entities/Article/model/slices/addCommentFormSlice'",
      errors: [
        {
          message: "В рамках одного слайса все пути должны быть относительными",
        },
      ],
    },
  ],
});

```

4) В rules при инициализации плагина прописываем:
```ts
    'fsd-checker-by-adelkkaa/fsd-path-checker': ['error', { alias: '@' }],
```

### Добавление fix свойства

Само правило выглядит следующим образом:
```js
/**
 * @fileoverview desc
 * @author adelkkaa
 */
"use strict";

const { isPathRelative } = require("../helpers");
const micromatch = require("micromatch");
const path = require("path");

//------------------------------------------------------------------------------
// Rule Definition
//------------------------------------------------------------------------------

/** @type {import('eslint').Rule.RuleModule} */
module.exports = {
  meta: {
    type: null, // `problem`, `suggestion`, or `layout`
    docs: {
      description: "descr",
      category: "Fill me in",
      recommended: false,
      url: null, // URL to the documentation page for this rule
    },
    fixable: null, // Or `code` or `whitespace`
    schema: [
      {
        type: "object",
        properties: {
          alias: {
            type: "string",
          },
          testFilesPatterns: {
            type: "array",
          },
        },
      },
    ],
  },

  create(context) {
    const { alias = "", testFilesPatterns = [] } = context.options[0] ?? {};

    const checkingLayers = {
      entities: "entities",
      features: "features",
      pages: "pages",
      widgets: "widgets",
    };

    return {
      ImportDeclaration(node) {
        const value = node.source.value;
        const importTo = alias ? value.replace(`${alias}/`, "") : value;

        if (isPathRelative(importTo)) {
          return;
        }

        // [entities, article, model, types]
        const segments = importTo.split("/");
        const layer = segments[0];

        if (!checkingLayers[layer]) {
          return;
        }

        const isImportNotFromPublicApi = segments.length > 2;
        // [entities, article, testing]
        const isTestingPublicApi =
          segments[2] === "testing" && segments.length < 4;

        if (isImportNotFromPublicApi && !isTestingPublicApi) {
          context.report(
            node,
            "Абсолютный импорт разрешен только из Public API (index.ts)"
          );
        }

        if (isTestingPublicApi) {
          const currentFilePath = context.getFilename();
          const normalizedPath = path.toNamespacedPath(currentFilePath);

          const isCurrentFileTesting = testFilesPatterns.some((pattern) =>
            micromatch.isMatch(normalizedPath, pattern)
          );

          if (!isCurrentFileTesting) {
            context.report(
              node,
              "Тестовые данные необходимо импортировать из publicApi/testing.ts"
            );
          }
        }
      },
    };
  },
};

```

Так выглядит правило с объявленным фиксом:
```js
/**
 * @fileoverview desc
 * @author adelkkaa
 */
"use strict";

const { isPathRelative } = require("../helpers");
const micromatch = require("micromatch");
const path = require("path");

//------------------------------------------------------------------------------
// Rule Definition
//------------------------------------------------------------------------------

const PUBLIC_ERROR = "PUBLIC_ERROR";
const TESTING_PUBLIC_ERROR = "TESTING_PUBLIC_ERROR";

/** @type {import('eslint').Rule.RuleModule} */
module.exports = {
  meta: {
    type: null, // `problem`, `suggestion`, or `layout`
    docs: {
      description: "descr",
      category: "Fill me in",
      recommended: false,
      url: null, // URL to the documentation page for this rule
    },
    fixable: "code", // Or `code` or `whitespace`
    messages: {
      [PUBLIC_ERROR]:
        "Абсолютный импорт разрешен только из Public API (index.ts)",
      [TESTING_PUBLIC_ERROR]:
        "Тестовые данные необходимо импортировать из publicApi/testing.ts",
    },
    schema: [
      {
        type: "object",
        properties: {
          alias: {
            type: "string",
          },
          testFilesPatterns: {
            type: "array",
          },
        },
      },
    ],
  },

  create(context) {
    const { alias = "", testFilesPatterns = [] } = context.options[0] ?? {};

    const checkingLayers = {
      entities: "entities",
      features: "features",
      pages: "pages",
      widgets: "widgets",
    };

    return {
      ImportDeclaration(node) {
        const value = node.source.value;
        const importTo = alias ? value.replace(`${alias}/`, "") : value;

        if (isPathRelative(importTo)) {
          return;
        }

        // [entities, article, model, types]
        const segments = importTo.split("/");
        const layer = segments[0];
        const slice = segments[1];


        if (!checkingLayers[layer]) {
          return;
        }

        const isImportNotFromPublicApi = segments.length > 2;
        // [entities, article, testing]
        const isTestingPublicApi =
          segments[2] === "testing" && segments.length < 4;

        if (isImportNotFromPublicApi && !isTestingPublicApi) {
          context.report({
            node,
            messageId: PUBLIC_ERROR,
            fix: (fixer) => {
              return fixer.replaceText(
                node.source,
                `'${alias}/${layer}/${slice}'`
              );
            },
          });
        }

        if (isTestingPublicApi) {
          const currentFilePath = context.getFilename();
          const normalizedPath = path.toNamespacedPath(currentFilePath);

          const isCurrentFileTesting = testFilesPatterns.some((pattern) =>
            micromatch.isMatch(normalizedPath, pattern)
          );

          if (!isCurrentFileTesting) {
            context.report({
              node,
              messageId: TESTING_PUBLIC_ERROR,
            });
          }
        }
      },
    };
  },
};

```

В данном случае просто меняем наполнение импорта при помощи fixer.replaceText

```js
context.report({
            node,
            messageId: PUBLIC_ERROR,
            fix: (fixer) => {
              return fixer.replaceText(
                node.source,
                `'${alias}/${layer}/${slice}'`
              );
            },
          });
```