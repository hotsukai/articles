---
title: "Snippets of Next.js written comfortably in VS Code"
topics: ['nextjs','react','vscode']
published: true
---

## Goal of This Article
Make it easier by making snippets of what you always write when you create a new component.



## VS Code allows you to register snippets
### There are two types.
- Set up the snippet in VS Code as a whole from the command palette.([Reference](https://code.visualstudio.com/docs/editor/userdefinedsnippets#_create-your-own-snippets))
- To use snippets per project by putting file names matching `*.code-snippets` in the `.vscode` directory.([Reference](https://code.visualstudio.com/updates/v1_28#_project-level-snippets))
  - When configuring on a per project, you can use the `scope` property to specify the language to be used.

### Amazingly multifunctional!
You can do a lot of things.
~~It's a pain to write~~ I can't write it all, so please go to [Documentation](https://code.visualstudio.com/docs/editor/userdefinedsnippets#_snippet-syntax) for details.
As an example
- Embed file names, directory names, etc. (regular expressions can also be used)
  - You can select default values, etc.
- You can specify the cursor position after using a snippet.

## Snippets I use in Next.js
### Create a Function component template with the same name as the file name

```json
{
  "new Function Component": {
    "prefix": "FC", // ⬅ String to be entered into the editor for snippet call
    "body": [ //⬅ The string to be inserted. To enter multiple lines, write an array.
      "import { FC } from 'react';",
      "", 
      "type Props = {};",
      "const ${1:$TM_FILENAME_BASE}: FC<Props> = ({}) => {",
      "  return <></>;",
      "};",
      "export default ${1:$TM_FILENAME_BASE};"
    ],
    "description": "Template of new FC",
  }
}
```

When the filename with the snippet is `Card.tsx`
The output is
```tsx
import { FC } from 'react';

type Props = {};
const Card: FC<Props> = ({}) => {
  return <></>;
};
export default Card;
```
File name is used as the component name.

### Create a NextPage template with the same name as the directory name.
- I want to create a NextPage template to be placed in `/pages/**/<PAGE_NAME>/Index.tx`.
- I want the name to be `<PAGE_NAME>`.

```json
{
  "new Next Page": {
    "prefix": "nextPage",
    "body": [
      "import type { NextPage } from 'next';",
      /* These lines are set differently for each project */
      "import HeadWrapper from '@/components/layout/HeadWrapper';", 
      "import Layout from '@/components/layout/Layout';",
      "import NeedLogin from '@/components/layout/NeedLogin';",
      "",
      /* POINT① */
      "const ${TM_DIRECTORY/.*\\/(.)(.+)$/${1:/upcase}$2/}: NextPage = () => {", 
      "  return (",
      "    <HeadWrapper>",
      "      <NeedLogin>",
      "        <Layout>",
      "          <main></main>",
      "        </Layout>",
      "      </NeedLogin>",
      "    </HeadWrapper>",
      "  );",
      "};",
      "",
      "export default ${TM_DIRECTORY/.*\\/(.)(.+)$/${1:/upcase}$2/};"
    ],
    "scope": "typescriptreact"
  }
}
```
In `POINT①`, the following is done.
1. since `TM_DIRECTORY` is an absolute path, use a regular expression to get the directory name where the file is located
2. capitalize the first letter of the directory name

The output will be the following, When the path of the file using the snippet is `pages/**/mypage/Index.tsx`.

```tsx
import type { NextPage } from 'next';
import HeadWrapper from '@/components/layout/HeadWrapper';
import Layout from '@/components/layout/Layout';
import NeedLogin from '@/components/layout/NeedLogin';

const Mypage: NextPage = () => {
  return (
    <HeadWrapper>
      <NeedLogin>
        <Layout>
          <main></main>
        </Layout>
      </NeedLogin>
    </HeadWrapper>
  );
};

export default Mypage;
```

## Snippets extension already existed😇.
The React snippet extension was already in the [marketplace](https://marketplace.visualstudio.com/items?itemName=dsznajder.es7-react-js-snippets).

However, as for me.
- The snippets are hard to remember.
- It defaults to `div` instead of Fragment.
- Props and type are not automatically attached.
- The component name is always a file name, not a directory name.
- ~~I'm more attached to snippets that I created myself.

I will continue to use the original snippet for these reasons.

Translated with www.DeepL.com/Translator (free version)