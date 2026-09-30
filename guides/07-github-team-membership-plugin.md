# GitHub Team Membership Plugin

## What this achieves
A Backstage Software Template where a user picks a GitHub organization, then a team inside that org (both dropdowns populate live from the GitHub API), chooses "add" or "remove", enters a GitHub username, and the backend adds or removes that user from the team via the GitHub API. Org and team lists are always live — no hardcoding, no manual updates when new orgs/teams are created.

## Prerequisites
- A working Backstage app (Node, Yarn set up, `yarn start` runs cleanly)
- GitHub auth already configured
- A GitHub Personal Access Token with the `admin:org` scope

## Architecture (3 pieces)
1. **Backend module** — a custom Scaffolder action (`github:team:membership`) that does the actual add/remove.
2. **Backend plugin** — two REST routes (`/orgs`, `/orgs/:org/teams`) that return live data from GitHub.
3. **Frontend field extensions** — two React dropdowns (`OrgPicker`, `TeamPicker`) that call those routes, used inside the template YAML via `ui:field`.

---

## Part 1 — Backend module: the scaffolder action

### Create it
```powershell
yarn new
```
Choose **backend-plugin-module**, plugin ID `scaffolder`, module ID `github-org`.

### Install dependency
```powershell
yarn --cwd plugins/scaffolder-backend-module-github-org add @octokit/rest
```

### `plugins/scaffolder-backend-module-github-org/src/actions/githubTeamMembership.ts`
```typescript
import { createTemplateAction } from '@backstage/plugin-scaffolder-node';
import { Octokit } from '@octokit/rest';
import { z } from 'zod';

export const createGithubTeamMembershipAction = () => {
  return createTemplateAction({
    id: 'github:team:membership',
    schema: {
      input: z.object({
        org: z.string().describe('The GitHub organization'),
        team: z.string().describe('The team slug'),
        username: z.string().describe('The GitHub username to add/remove'),
        operation: z
          .enum(['add', 'remove'])
          .describe('Whether to add or remove the user'),
        token: z.string().describe('GitHub token with admin:org scope'),
      }),
    },
    async handler(ctx) {
      const { org, team, username, operation, token } = ctx.input;
      const octokit = new Octokit({ auth: token });

      if (operation === 'add') {
        await octokit.teams.addOrUpdateMembershipForUserInOrg({
          org,
          team_slug: team,
          username,
        });
        ctx.logger.info(`Added ${username} to ${org}/${team}`);
      } else {
        await octokit.teams.removeMembershipForUserInOrg({
          org,
          team_slug: team,
          username,
        });
        ctx.logger.info(`Removed ${username} from ${org}/${team}`);
      }
    },
  });
};
```

### `plugins/scaffolder-backend-module-github-org/src/module.ts`
```typescript
import { createBackendModule } from '@backstage/backend-plugin-api';
import { scaffolderActionsExtensionPoint } from '@backstage/plugin-scaffolder-node';
import { createGithubTeamMembershipAction } from './actions/githubTeamMembership';

export const scaffolderModuleGithubOrg = createBackendModule({
  pluginId: 'scaffolder',
  moduleId: 'github-org',
  register(reg) {
    reg.registerInit({
      deps: {
        scaffolderActions: scaffolderActionsExtensionPoint,
      },
      async init({ scaffolderActions }) {
        scaffolderActions.addActions(createGithubTeamMembershipAction());
      },
    });
  },
});
```
`yarn new` wires this module into `packages/backend/src/index.ts` automatically. Confirm one single line like this exists there (not duplicated):
```typescript
backend.add(import('@internal/backstage-plugin-scaffolder-backend-module-github-org'));
```

---

## Part 2 — Backend plugin: live org/team API routes

### Create it
```powershell
yarn new
```
Choose **backend-plugin**, ID `github-org`.

### Install dependencies
```powershell
yarn --cwd plugins/github-org-backend add @octokit/rest express express-promise-router
yarn --cwd plugins/github-org-backend add -D @types/express
```

### `plugins/github-org-backend/src/plugin.ts`
```typescript
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { Octokit } from '@octokit/rest';
import express from 'express';
import Router from 'express-promise-router';

export const githubOrgPlugin = createBackendPlugin({
  pluginId: 'github-org',
  register(reg) {
    reg.registerInit({
      deps: {
        httpRouter: coreServices.httpRouter,
        config: coreServices.rootConfig,
        logger: coreServices.logger,
      },
      async init({ httpRouter, config, logger }) {
        const githubIntegrations = config.getConfigArray('integrations.github');
        const token = githubIntegrations[0].getString('token');
        const octokit = new Octokit({ auth: token });

        const router = Router();
        router.use(express.json());

        router.get('/orgs', async (_req, res) => {
          const { data } = await octokit.orgs.listForAuthenticatedUser({
            per_page: 100,
          });
          res.json(data.map(o => o.login));
        });

        router.get('/orgs/:org/teams', async (req, res) => {
          const { org } = req.params;
          const { data } = await octokit.teams.list({
            org,
            per_page: 100,
          });
          res.json(data.map(t => ({ name: t.name, slug: t.slug })));
        });

        httpRouter.use(router);
        logger.info('github-org backend routes registered');
      },
    });
  },
});
```

### `plugins/github-org-backend/src/index.ts`
```typescript
export { githubOrgPlugin as default } from './plugin';
```

Confirm exactly one line registers this in `packages/backend/src/index.ts`:
```typescript
backend.add(import('@internal/backstage-plugin-github-org-backend'));
```
(Check the real package name in `plugins/github-org-backend/package.json`'s `"name"` field — use that exact string.)

---

## Part 3 — Frontend: live dropdown field extensions

### `packages/app/src/scaffolder/OrgPicker/OrgPickerExtension.tsx`
```tsx
import React, { useEffect, useState } from 'react';
import { FieldExtensionComponentProps } from '@backstage/plugin-scaffolder-react';
import FormControl from '@material-ui/core/FormControl';
import InputLabel from '@material-ui/core/InputLabel';
import Select from '@material-ui/core/Select';
import MenuItem from '@material-ui/core/MenuItem';
import { useApi, fetchApiRef, discoveryApiRef } from '@backstage/core-plugin-api';

export const OrgPicker = ({
  onChange,
  formData,
  required,
}: FieldExtensionComponentProps<string>) => {
  const { fetch } = useApi(fetchApiRef);
  const discoveryApi = useApi(discoveryApiRef);
  const [orgs, setOrgs] = useState<string[]>([]);

  useEffect(() => {
    (async () => {
      const baseUrl = await discoveryApi.getBaseUrl('github-org');
      const res = await fetch(`${baseUrl}/orgs`);
      const data = await res.json();
      setOrgs(data);
    })().catch(() => setOrgs([]));
  }, [fetch, discoveryApi]);

  return (
    <FormControl margin="normal" required={required} fullWidth>
      <InputLabel htmlFor="orgPicker">Organization</InputLabel>
      <Select
        id="orgPicker"
        value={formData || ''}
        onChange={e => onChange(e.target.value as string)}
      >
        {orgs.map(org => (
          <MenuItem key={org} value={org}>
            {org}
          </MenuItem>
        ))}
      </Select>
    </FormControl>
  );
};
```

### `packages/app/src/scaffolder/OrgPicker/index.ts`
```typescript
import { FormFieldBlueprint, createFormField } from '@backstage/plugin-scaffolder-react/alpha';
import { OrgPicker } from './OrgPickerExtension';

export const OrgPickerFieldExtension = FormFieldBlueprint.make({
  name: 'OrgPicker',
  params: {
    field: async () =>
      createFormField({
        name: 'OrgPicker',
        component: OrgPicker,
      }),
  },
});
```

### `packages/app/src/scaffolder/TeamPicker/TeamPickerExtension.tsx`
```tsx
import React, { useEffect, useState } from 'react';
import { FieldExtensionComponentProps } from '@backstage/plugin-scaffolder-react';
import FormControl from '@material-ui/core/FormControl';
import InputLabel from '@material-ui/core/InputLabel';
import Select from '@material-ui/core/Select';
import MenuItem from '@material-ui/core/MenuItem';
import { useApi, fetchApiRef, discoveryApiRef } from '@backstage/core-plugin-api';

export const TeamPicker = ({
  onChange,
  formData,
  required,
  formContext,
}: FieldExtensionComponentProps<string>) => {
  const { fetch } = useApi(fetchApiRef);
  const discoveryApi = useApi(discoveryApiRef);
  const [teams, setTeams] = useState<{ name: string; slug: string }[]>([]);
  const selectedOrg = (formContext?.formData as any)?.org;

  useEffect(() => {
    if (!selectedOrg) {
      setTeams([]);
      return;
    }
    (async () => {
      const baseUrl = await discoveryApi.getBaseUrl('github-org');
      const res = await fetch(`${baseUrl}/orgs/${selectedOrg}/teams`);
      const data = await res.json();
      setTeams(data);
    })().catch(() => setTeams([]));
  }, [fetch, discoveryApi, selectedOrg]);

  return (
    <FormControl margin="normal" required={required} fullWidth disabled={!selectedOrg}>
      <InputLabel htmlFor="teamPicker">Team</InputLabel>
      <Select
        id="teamPicker"
        value={formData || ''}
        onChange={e => onChange(e.target.value as string)}
      >
        {teams.map(team => (
          <MenuItem key={team.slug} value={team.slug}>
            {team.name}
          </MenuItem>
        ))}
      </Select>
    </FormControl>
  );
};
```

### `packages/app/src/scaffolder/TeamPicker/index.ts`
```typescript
import { FormFieldBlueprint, createFormField } from '@backstage/plugin-scaffolder-react/alpha';
import { TeamPicker } from './TeamPickerExtension';

export const TeamPickerFieldExtension = FormFieldBlueprint.make({
  name: 'TeamPicker',
  params: {
    field: async () =>
      createFormField({
        name: 'TeamPicker',
        component: TeamPicker,
      }),
  },
});
```

### Register both in `packages/app/src/App.tsx`
```tsx
import { OrgPickerFieldExtension } from './scaffolder/OrgPicker';
import { TeamPickerFieldExtension } from './scaffolder/TeamPicker';

const scaffolderFieldExtensions = createFrontendModule({
  pluginId: 'scaffolder',
  extensions: [OrgPickerFieldExtension, TeamPickerFieldExtension],
});

export default createApp({
  features: [
    // ...existing features
    scaffolderFieldExtensions,
  ],
});
```

---

## Part 4 — The template YAML

`github-team-membership-template.yaml`:
```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: github-team-membership
  title: Manage GitHub Team Membership
  description: Add or remove a user from a GitHub org team
spec:
  owner: guests
  type: service
  parameters:
    - title: Select org and team
      properties:
        org:
          title: Organization
          type: string
          ui:field: OrgPicker
        team:
          title: Team
          type: string
          ui:field: TeamPicker
    - title: Choose action
      properties:
        operation:
          title: Operation
          type: string
          enum: ['add', 'remove']
          enumNames: ['Add user', 'Remove user']
          default: 'add'
        username:
          title: GitHub username
          type: string
  steps:
    - id: manage-membership
      name: Update team membership
      action: github:team:membership
      input:
        org: ${{ parameters.org }}
        team: ${{ parameters.team }}
        username: ${{ parameters.username }}
        operation: ${{ parameters.operation }}
        token: ${{ secrets.GITHUB_TOKEN }}
  output:
    text:
      - title: Result
        content: 'Done'
```

Register it in `app-config.yaml`'s `catalog.locations`:
```yaml
    - type: file
      target: ../../github-team-membership-template.yaml
      rules:
        - allow: [Template]
```

---

## Done
Restart with `yarn start`, open `localhost:3000/create`, find "Manage GitHub Team Membership" — Organization and Team dropdowns populate live from GitHub, pick add/remove, enter a username, run it.

