# Cloud Pi Native plugin: Hello World API

La console [Cloud π Native](https://github.com/cloud-pi-native/console) a un système de hook permettant d'étendre ses fonctionnalités via des plugins.

Le dépôt [helloworld](https://github.com/cloud-pi-native/console-plugin-helloworld) décrit la structure attendue ainsi que les différents hooks auxquels il est possible de se rattacher.

Il est aussi possible d'écrire un plugin étendant les fonctionnalités de la console rendant des services aux autres plugins.

Un exemple de ce type de plugin est celui gérant un Hashicorp Vault. Ce plugin permet aux autres d'écrire/lire leurs secrets afin de pouvoir se connecter à différents webservice.

Le plugin exposant une API n'est pas appelé directement par les autres plugins, il faut à chaque fois passer par le PluginManager (gestionnaire des hooks) qui sert de place centrale.

## Installation

La console Cloud Pi Native et les plugins associés sont codés en JavaScript/TypeScript.

### Dépendances
Les dépendances suivantes sont nécessaires:
- "@cpn-console/hooks": Permet de s'inscrire à des hooks
- "@cpn-console/shared": Fonctions utilitaires

Les dépendances de développement suivantes sont optionnelles:
- "@cpn-console/eslint-config": configuration pour le linter
- "@cpn-console/ts-config": configuration pour typescript

## Développement
Dans cet exemple, le code est divisé en 2 fichiers:
- index.ts: point d'entrée du plugin contenant des informations à propos de ce dernier
- class.js: logique métier du plugin API


### Point d'entrée
Le fichier index.ts sert de point d'entrée pour le chargement du plugin et l'inscription aux hooks disponibles.

```ts
import type { DefaultArgs, Plugin, Project, ProjectLite, ServiceInfos } from '@cpn-console/hooks'
import { archiveDsoProject } from './functions.js'
import infos from './infos.js'
import { VaultProjectApi } from './class.js'

const onlyApi = { api: (project: ProjectLite) => new VaultProjectApi(project) }

export const vaultUrl = process.env.VAULT_URL

const infos: ServiceInfos = {
  name: 'vault',
  title: 'Vault',
}

export const plugin: Plugin = {
  infos,
  subscribedHooks: {
    getProjectSecrets: onlyApi,
    upsertProject: onlyApi,
    deleteProject: {
      ...onlyApi,
      steps: { post: archiveDsoProject }, // Destroy all secrets for project
    },
  },
  monitor,
}

declare module '@cpn-console/hooks' {
  interface HookPayloadApis<Args extends DefaultArgs> {
    vault: Args extends (ProjectLite | Project)
    ? VaultProjectApi
    : undefined
  }
}
```

## Déclaration de l'API

Pour exposer des méthodes au titre d'API, écrire une classe qui étend `PluginApi`:

Dans cet exemple, la classe `VaultProjectApi` expose les méthodes suivantes aux autres plugins:
- list
- read
- write
- destroy

```ts
import { PluginApi, ProjectLite } from '@cpn-console/hooks'

type readOptions = {
  throwIfNoEntry: boolean
}

export class VaultProjectApi extends PluginApi {
    constructor (project: ProjectLite) {
        super()
    }

    private async getToken () {
        ...
    }

    public async list (path: string = '/'): Promise<string[]> {
        ...
    }

    public async read (path: string = '/', options: readOptions = { throwIfNoEntry: true }) {
        ...
    }

    public async write (body: object, path: string = '/') {
        ...
    }

    public async destroy (path: string = '/') {
        ...
    }
}
```

## Utilisation des méthodes par d'autres plugins

### Développement

#### Typage
Même s'il faut passer par le PluginManager pour avoir accès aux méthodes exposées par un plugin de type API, il est possible de l'importer afin d'avoir accès aux types/méthodes exposées.

Dans le fichier `package.json`:
```json
"devDependencies": {
    "@cpn-console/eslint-config": "^1.0.0",
    "@cpn-console/vault-plugin": "^2.0.1",
    "@cpn-console/ts-config": "^1.1.0",
  },
```

Dans le fichier `src/end.d.ts`: 
```
<reference types="@cpn-console/vault-plugin/types/index.d.ts" />
```

### Appel API

Nous allons maintenant voir comment faire appels aux méthodes exposées par le plugin vault.

Dans la fonction callback de son plugin, le payload à une propriété `apis` listant les apis et les méthodes disponibles.

Exemple avec l'api vault:
```ts
export const plugin: Plugin = {
  infos,
  subscribedHooks: {
    upsertProject: { steps: { post: createDsoProject } },
  },
}
```

```ts
export const createDsoProject: StepCall<Project> = async (payload) => {
  try {
    const vaultApi = payload.apis.vault

    const vaultRegistrySecret = await vaultApi.read(...)
    
    vaultApi.write(...)
    await vaultApi.write(...)

    return {
      status: {
        result: 'OK',
        message: 'Created',
      },
      result: {
        project: projectCreated,
      },
    }
  } catch (error) {
    return {
      error: parseError(error),
      status: {
        result: 'KO',
        // @ts-ignore prévoir une fonction générique
        message: error.message,
      },
    }
  }
}
```