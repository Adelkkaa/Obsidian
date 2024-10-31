> Для автоматизированного рефакторинга архитектуры проекта, можно использовать ts-morph

### Пример подстановки alias

Было `import { Any } from 'shared/ui/Any'
Стало `import { Any } from '@/shared/ui/Any'

```ts
import { Project } from 'ts-morph';

const project = new Project({});

project.addSourceFilesAtPaths('src/**/*.ts');
project.addSourceFilesAtPaths('src/**/*.tsx');

const files = project.getSourceFiles();

function isAbsolute(value: string) {
    const layers = ['app', 'shared', 'entities', 'features', 'widgets', 'pages'];
    return layers.some((layer) => value.startsWith(layer));
}

files.forEach((sourceFile) => {
    const importDeclarations = sourceFile.getImportDeclarations();
    importDeclarations.forEach((importDeclaration) => {
        const value = importDeclaration.getModuleSpecifierValue();

        if (isAbsolute(value)) {
            importDeclaration.setModuleSpecifier(`@/${value}`);
        }
    });
});

project.save();

```

Запускается данный скрипт при помощи `ts-node ./path/script_name.extension`

