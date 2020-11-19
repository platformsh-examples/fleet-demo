# Platform.sh API Fleet Management Demo 

## Requirements

- An API token, which can be [created from your Platform.sh account settings](https://docs.platform.sh/development/cli/api-tokens.html#obtaining-a-token), copied into a local file called `token`. This file will not be committed.
- [`jq`](https://stedolan.github.io/jq/download/)

## Purpose

This demo wraps around the Platform.sh API to manage a fleet of projects more easily. Here, a "fleet" is defined by an upstream repository, it's profile name, and the prefix string that each project on Platform.sh will have as the first part of its `project_title`. In this way, it could very well be modified to allow you to manage multiple fleets, but it is a simple "one fleet demo" for now. 

The demo pulls together multiple concepts

Additionally, the demo in its current form does not lump actions across the fleet, although that might be desirable. By this I mean that triggering a source operation on a single project can be run with this tooling, but not on *every* project in the fleet. Maybe we'll add it later, but I hope that you can at least see from a single example how fleet-wide actions could be added. If you would like to see such a fleet-wide action as an example, you can take a look at `./fleet.sh cleanup`, which will delete every project in the fleet (since this is a demo after all, and we should really cleanup after ourselves).

All of the above attributes are defined in `config.json`.

```json
// config.json
{
    "region": "us-4.platform.sh",
    "plan": "development",
    "project_prefix": "Directus demo | ",
    "upstream": {
        "repository": "https://github.com/chadwcarlson/directus-next.git",
        "profile": "Directus Demo"
    },
    "activity_scripts": [
        "slack.js"
    ],
    "updates": {
        "environment_name": "directus-updates",
        "title": "Directus updates"
    }
}
```

Hopefully, this demo is useful to what you're trying to build. It makes some assumptions of scope and tooling, but as it exposes the Platform.sh API directly it is mean primarily to teach rather that supply anything close to production ready. 

For example, it's highly unlikely that you'll interact with the API directly, unless you are using a language for which we don't already have an API client for. In those cases, I encourage you to look at our API clients in order to integrate these actions into whatever fleet management dashboard you're building.

- [platformsh-client-php: PHP API Client library](https://github.com/platformsh/platformsh-client-php)
- [platformsh-client-js: JS API Client library](https://github.com/platformsh/platformsh-client-js)

Additionally, you're likely going to want to develop some kind of GUI to manage your fleet rather than the "pseudo-CLI" shown here. We have created a very basic reference implementation of such a GUI called [Admiral](), a Symfony project that uses the Platform.sh PHP Client. 

- [Admiral on GitHub](https://github.com/platformsh/admiral)
- [Blog - Source Operations: automate maintenance, from single sites to fleets](https://platform.sh/blog/2019/website-fleet/automating-updates-source-operations/)
- [Blog - Admiral: How to keep your fleet afloat](https://platform.sh/blog/2019/website-fleet/admiral-an-example-of-website-fleet-automation/)

## Using the demo

### Getting started

- First, retrieve an API Token [from your Platform.sh account settings](https://docs.platform.sh/development/cli/api-tokens.html#obtaining-a-token). Copy and paste that token into a file called `demo/token`. This file is not committed, but the key will be used in multiple places across the demo. 
- Take a look at `config.json`. This object controls how each new project in the fleet will be initialized. 
- Run `./fleet.sh list`. You should see no projects in your fleet. 

### Setting up projects in the fleet

- Run `./fleet.sh new "Client 1"` to create a project in the fleet. This will only take a moment. 
- Run `./fleet.sh list`. You should now see the project listed for your fleet. Visit the management console url listed for that command. 
- Run `./fleet.sh init SUBSCRIPTIONID`. Initialize Master environment on the current project with the upstream repository and profile name.
- Run `./fleet.sh initVars SUBSCRIPTIONID`. Initialize *project*-level environment variables according to the fleet configuration.
- Run `./fleet.sh redeploy SUBCRIPTIONID`. This will redeploy the Master environment, giving you a freshly initialized project now with access to fleet.
- Run `./fleet.sh createUpdatesBranch SUBSCRIPTIONID`. This sets up a dedicated branch for dependency and upstream updates on the project.
- Run `./fleet.sh addActivityScript SUBCRIPTIONID`. This adds the local `slack.js` activity script to the project, which will send slack messages from the dedicated updates branch when a source operation has completed. It's at this point where you can enter the Webhook URL of the Slack channel you want notifications to be sent to, which will be added as a *project*-level environment variable.
- Run `./fleet.sh redeploy SUBCRIPTIONID directus-updates` to update the evironments's access to the new environment variable.

> **Note:** 
>
> Repeat the steps above for each additional project you want to add to the fleet.

### Managing projects in the fleet



## Important concepts and resources

- [Platform.sh API documentation](https://api.platform.sh/docs/)
- [OpenAPI Specification for the Platform.sh API](https://api.platform.sh/docs/openapispec-platformsh.json)
- [Source Operations](https://docs.platform.sh/configuration/app/source-operations.html)
- [Activity scripts](https://docs.platform.sh/integrations/activity.html)
- [Activity scripts example: Slack integration](https://docs.platform.sh/integrations/activity/slack.html)
- [YouTube demo: Automating Composer updates with Source Operations and Activity Scripts](https://www.youtube.com/watch?v=MILHG9OqhmE&t=226s)
- [platformsh-client-php: PHP API Client library](https://github.com/platformsh/platformsh-client-php)
- [platformsh-client-js: JS API Client library](https://github.com/platformsh/platformsh-client-js)
- [Admiral: a Fleet Management GUI Reference in Symfony](https://github.com/platformsh/admiral)
- [Blog - Source Operations: automate maintenance, from single sites to fleets](https://platform.sh/blog/2019/website-fleet/automating-updates-source-operations/)
- [Blog - Admiral: How to keep your fleet afloat](https://platform.sh/blog/2019/website-fleet/admiral-an-example-of-website-fleet-automation/)

## Limitations

### Project ownership and client customization

In this demo, *your* API token is used to create new projects, and by default those projects will be created on your account (rather than, say, a client's account). In order to enable source operations (which contain CLI commands to sync data & code or take backups prior to the main operation itself), *your* API token is added as an environment variable at the project level. This combination of choices covers a "fully managed" fleet use case, where clients manage their API entirely through the UI, and do not have a separate repository that they push updates to, nor are they assumed to have access to the project via the management console or through SSH.  

Maintaining a fleet where clients have access to their projects via the management console, CLI, and SSH comes with additional considerations not included in this demo:

- Placing the project on the client's account
- Using the client's API token for in-container CLI commands (not exposing Fleet managers token)
- Source integrations: Allowing for client commits while maintaining upstream repo functionality

### A backend without a frontend

In this demo, the upstream repository is a simple, single application Directus site. That is, it's only the backend API. This, of course, doesn't have to be the case. You could very easily manage a fleet where a [multiple application](https://docs.platform.sh/configuration/app/multi-app.html#applicationsyaml) repository is used as each project's upstream. We have a number of examples from our templates that you can check out as a reference:

- [Gatsby frontend, Drupal backend](https://github.com/platformsh-templates/gatsby-drupal)
- [Gatsby frontend, Strapi backend](https://github.com/platformsh-templates/gatsby-strapi)
- [Gatsby frontend, WordPress backend](https://github.com/platformsh-templates/gatsby-wordpress)

## API

Running `./fleet.sh`:

- `list`: Lists all projects in the fleet.
- `new PROJECT_TITLE`: Will create a new project with the name `Directus | PROJECT_TITLE`.
- `delete SUBSCRIPTION_ID`: Deletes project from the fleet, first providing a list to retrieve `SUBSCRIPTION_ID`.
- `init SUBSCRIPTION_ID`: Initializes project with "Fleet template", first providing a list to retrieve `SUBSCRIPTION_ID`.
- `initVars SUBSCRIPTION_ID`: Adds fleet environment variables to a project.
- `redeploy SUBSCRIPTION_ID`: Saves changes from the initialization steps, completing a new running project.
- `cleanup`: When you are finished with the demo, you can use this command to delete the demo fleet from Platform.sh.

