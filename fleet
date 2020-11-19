#!/usr/bin/env bash

############################################## CONFIGURATION ##############################################

CONFIG=config.json

PROJECT_PREFIX=$(cat $CONFIG | jq -r '.project_prefix')
PSH_REGION=$(cat $CONFIG | jq -r '.region')
PSH_PLAN=$(cat $CONFIG | jq -r '.plan')

FLEET_PROFILE=$(cat $CONFIG | jq -r '.upstream.profile')
FLEET_REPO=$(cat $CONFIG | jq -r '.upstream.repository')

FLEET_UPDATES_BRANCH=$(cat $CONFIG | jq -r '.updates.environment_name')
FLEET_UPDATES_TITLE=$(cat $CONFIG | jq -r '.updates.title')

############################################## AUTHENTICATION ##############################################

# To make it easy, just save an API token in an uncommitted file called 'token'.
#   See https://docs.platform.sh/development/cli/api-tokens.html#obtaining-a-token.
PLATFORMSH_API_TOKEN=$(cat token)
HEADER="Content-Type: application/json"

# Get a fresh OAuth token, since they expire.
refreshOAuthToken() {
    curl -s -X POST \
        -H "$HEADER" \
        -d '{
            "client_id": "platform-api-user",
            "grant_type": "api_token",
            "api_token": "'"$PLATFORMSH_API_TOKEN"'"
        }' \
        https://accounts.platform.sh/oauth2/token | jq -r '.access_token'
}

############################################## GETTING FLEET INFORMATION ##############################################

# Retrieves project information from subscription ID.
getProjectInfoFromSubscription() {
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    curl -s -X GET \
        -H "$HEADER" -H "$AUTH" \
        https://api.platform.sh/subscriptions/$1
}

# List projects in the fleet. 
list() {
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    PROJECTS=$(curl -s -X GET \
        -H "$HEADER" -H "$AUTH" \
        https://api.platform.sh/subscriptions ) 
    
    NUMPROJECTS=$(echo $PROJECTS | jq -r --arg PROJECT_PREFIX "$PROJECT_PREFIX" '.subscriptions | to_entries[] | select(.value.project_title | contains($PROJECT_PREFIX))')
    if [[ $NUMPROJECTS ]]; then
        echo -e "\nProjects in your fleet:\n\nid\tproject_id\tproject_title\t\t\tproject_ui\n"
        echo $PROJECTS | jq -r --arg PROJECT_PREFIX "$PROJECT_PREFIX" '.subscriptions | to_entries[] | select(.value.project_title | contains($PROJECT_PREFIX)) | [ .value.id, .value.project_id, .value.project_title, .value.project_ui ] | join("\t") '
        echo -e "\n"
    else
        echo -e "\nYou're fleetless.\n"
    fi
}

############################################## PROJECT VARIABLES ##############################################

# Initialize fleet variables on project main function.
initVars() {
    if [[ $1 ]]; then
        addFleetVariables $1
    else
        initPrompt
    fi
}

# Adds collection of fleet variables.
addFleetVariables(){
    echo -e "\nAdding fleet environment variables to project..."
    # Add API token project variable
    addProjectVariable --id="$1" --key="PLATFORMSH_CLI_TOKEN" --value="$PLATFORMSH_API_TOKEN" --sensitive=true
    # Add Upstream repo environment variable
    addProjectVariable --id="$1" --key="UPSTREAM_REPO" --value="$FLEET_REPO" --sensitive=false
}

# Prompt to allow single subscription ID to be passed.
initVarsPrompt() {
    # Get subcription ID from the user.
    list
    read -p "Enter the subscription ID you'd like to initialize with fleet environment variables: "  id    
    echo -e "\nAdding fleet environment variables to project..."
    addFleetVariables $id
}

# Adds a single project-level variable to a project.
#   --id=SUBSCRIPTION ID (string)
#   --key=Variable key; `env:` prefix assumed and not required. (string)
#   --value=Variable's value. (string)
#   --sensitive (bool)
addProjectVariable() {
    # Get named arguments.
    for ARGUMENT in "$@"
    do
        KEY=$(echo $ARGUMENT | cut -f1 -d=)
        VALUE=$(echo $ARGUMENT | cut -f2 -d=)   
        case "$KEY" in
                --id)       ID=${VALUE} ;;
                --key)      ENVVAR_KEY=${VALUE} ;;
                --value)    ENVVAR_VALUE=${VALUE} ;;     
                --sensitive)    SENSITIVE=${VALUE} ;;   
                *)   
        esac    
    done
    # Get project ID from subscription ID.
    PROJECT_INFO=$(getProjectInfoFromSubscription $ID)
    PROJECT_TITLE=$(echo $PROJECT_INFO | jq -r '.project_title')
    PROJECT_ID=$(echo $PROJECT_INFO | jq -r '.project_id')
    PROJECT_UI=$(echo $PROJECT_INFO | jq -r '.project_ui')
    # Modify the variable key.
    VAR_KEY="env:$ENVVAR_KEY"
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    RESPONSE=$(curl -s -X POST \
        -H "$HEADER" -H "$AUTH" \
        -d '{
            "name": "'"$VAR_KEY"'",
            "value": "'"$ENVVAR_VALUE"'",
            "is_json": false,
            "is_sensitive": "'"$SENSITIVE"'",
            "visible_build": true,
            "visible_runtime": true
        }' \
        https://api.platform.sh/projects/$PROJECT_ID/variables)
}

############################################## NEW PROJECTS ##############################################

# Create a new subscription (project) in the fleet.
new() {
    if [ $# -eq 0 ]
    then
        echo -e "\nError: Give the project a name.\n\nExample:\n\n\t./fleet.sh create XXXXXX\n\nNote that the name 'demo' will become '$PROJECT_PREFIX-XXXXXX' on your Platform.sh account based on the configuration.\n"
    else
        PROJECT_TITLE="$PROJECT_PREFIX$1"
        AUTH="Authorization: Bearer $(refreshOAuthToken)"
        RESPONSE=$(curl -s -X POST \
            -H "$HEADER" -H "$AUTH" \
            -d '{
                "project_region": "'"$PSH_REGION"'",
                "project_title": "'"$PROJECT_TITLE"'",
                "plan": "'"$PSH_PLAN"'"
            }' \
            https://api.platform.sh/subscriptions)

        echo -e "\nCreate project ($PROJECT_PREFIX$1) on Platform.sh requested.\nGive it a moment, then run 'init' to initialize the project.\n"
    fi
}

# Allow user to pass subscription ID.
init() {
    if [[ $1 ]]; then
        initMaster $1
    else
        initPrompt
    fi
}

# Prompt to initialize subscription (project) Master environment with upstream repository and profile.
initPrompt() {
    # Get subcription ID from the user.
    list
    read -p "Enter the subscription ID you'd like to initialize with the upstream: "  id    
    read -p "You sure about that? [y|N]: " initializeCase
    case $initializeCase in  
        y|Y) initMaster $id;; 
        n|N|*) echo -e "\nAlright, I'll be here if you need me.\n" ;; 
    esac
}

# Initialize subscription (project) with the upstream repository and profile.
initMaster() {
    # Get project ID from subscription ID.
    PROJECT_INFO=$(getProjectInfoFromSubscription $1)
    PROJECT_TITLE=$(echo $PROJECT_INFO | jq -r '.project_title')
    PROJECT_ID=$(echo $PROJECT_INFO | jq -r '.project_id')
    PROJECT_UI=$(echo $PROJECT_INFO | jq -r '.project_ui')
    echo -e "\nInitializing master environment on project $PROJECT_TITLE ($PROJECT_ID)...\n"

    # Initialize the master environment with upstream profile.
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    RESPONSE=$(curl -s -X POST \
        -H "$HEADER" -H "$AUTH" \
        -d '{
            "profile": "'"$FLEET_PROFILE"'",
            "repository": "'"$FLEET_REPO"'"
        }' \
        https://api.platform.sh/projects/$PROJECT_ID/environments/master/initialize)

    # Return status.
    echo -e "\nMaster environment $PROJECT_TITLE ($PROJECT_ID) initialized:\n\tProfile:\t'$FLEET_PROFILE'\n\tRepository:\t$FLEET_REPO\n\nView it at $PROJECT_UI/master.\n"
}

############################################## DELETING PROJECTS ##############################################

# Delete a Platform.sh project.
deleteProject() {
    echo -e "\nDeleting subcription $1...\n"
    # Delete a project from a subscription ID.
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    curl -X DELETE \
        -H "$HEADER" -H "$AUTH" \
        https://api.platform.sh/subscriptions/$1 

    echo -e "\nProject deleted.\n"
}

# Show list of projects in fleet if subscription ID is not given.
deletePrompt() {
    list
    read -p "Enter the subscription ID you'd like to delete: "  id    
    read -p "You sure about that? It's irreversible. [y|N]: " deleteCase
    case $deleteCase in  
        y|Y) deleteProject $id;; 
        n|N|*) echo -e "\nAlright, I'll be here if you need me.\n" ;; 
    esac
}

# Main delete project function
delete () {
    if [[ $1 ]]; then
        deleteProject $1
    else
        deletePrompt
    fi
}

# Delete the whole fleet.
deleteFleet() {
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    PROJECTS=$(curl -s -X GET \
        -H "$HEADER" -H "$AUTH" \
        https://api.platform.sh/subscriptions ) 

    FLEET=$(echo $PROJECTS | jq -r --arg PROJECT_PREFIX "$PROJECT_PREFIX" '.subscriptions | to_entries[] | select(.value.project_title | contains($PROJECT_PREFIX)) | [ .value.id ]')
    echo $FLEET
    # my_string="Ubuntu;Linux Mint;Debian;Arch;Fedora"
    # IFS=';' read -ra my_array <<< "$my_string"
    # for i in "${my_array[@]}"
    # do
    #     echo $i
    # done
    # for PROJECT_TO_DELETE in "${FLEET[@]}"
    # do
    #     deleteProject $PROJECT_TO_DELETE
    #     # echo -e "\n$PROJECT_TO_DELETE"
    #     sleep 10
    # done
    # list
    # echo -e "\nThanks for trying out the demo! Go forth and Deploy Friday!\n"
}

# Function for cleaning up demo on Platform.sh.
cleanup() {
    list
    read -p "Are you sure you want to delete your fleet? No turning back from that, friend. [y|N]: " deleteCase
    case $deleteCase in  
        y|Y) deleteFleet ;; 
        n|N|*) echo -e "\nYeah, better not.\n" ;; 
    esac
}

############################################## ENVIRONMENT ACTIONS ##############################################

# Redeploy the Master environment for a given subscription (project).
redeploy() {
    # Master environment by default. Second argument for environment name.
    BRANCH="master"
    if [[ $2 ]]; then 
        unset $BRANCH && BRANCH=$2
    fi
    # Get project ID from subscription ID.
    PROJECT_INFO=$(getProjectInfoFromSubscription $1)
    PROJECT_TITLE=$(echo $PROJECT_INFO | jq -r '.project_title')
    PROJECT_ID=$(echo $PROJECT_INFO | jq -r '.project_id')
    PROJECT_UI=$(echo $PROJECT_INFO | jq -r '.project_ui')
    echo -e "\nRedploying $BRANCH environment on project $PROJECT_TITLE ($PROJECT_ID)...\n"

    # Redeploy the environment.
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    RESPONSE=$(curl -s -X POST \
        -H "$HEADER" -H "$AUTH" \
        https://api.platform.sh/projects/$PROJECT_ID/environments/$BRANCH/redeploy)
}

createUpdatesBranch() {
    # Get project ID from subscription ID.
    PROJECT_INFO=$(getProjectInfoFromSubscription $1)
    PROJECT_TITLE=$(echo $PROJECT_INFO | jq -r '.project_title')
    PROJECT_ID=$(echo $PROJECT_INFO | jq -r '.project_id')
    PROJECT_UI=$(echo $PROJECT_INFO | jq -r '.project_ui')
    echo -e "\nCreating fleet update environment on project $PROJECT_TITLE ($PROJECT_ID)...\n"
    # Initialize the master environment with upstream profile.
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    RESPONSE=$(curl -s -X POST \
        -H "$HEADER" -H "$AUTH" \
        -d '{
            "name": "'"$FLEET_UPDATES_BRANCH"'",
            "clone_parent": true,
            "title": "'"$FLEET_UPDATES_TITLE"'"
        }' \
        https://api.platform.sh/projects/$PROJECT_ID/environments/master/branch)
    # echo $RESPONSE | jq
}

getIntegrations() {
    PROJECT_INFO=$(getProjectInfoFromSubscription $1)
    PROJECT_TITLE=$(echo $PROJECT_INFO | jq -r '.project_title')
    PROJECT_ID=$(echo $PROJECT_INFO | jq -r '.project_id')
    PROJECT_UI=$(echo $PROJECT_INFO | jq -r '.project_ui')
    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    RESPONSE=$(curl -s -X GET \
        -H "$HEADER" -H "$AUTH" \
        https://api.platform.sh/projects/$PROJECT_ID/integrations)
    echo $RESPONSE | jq
}

addActivityScript() {
    PROJECT_INFO=$(getProjectInfoFromSubscription $1)
    PROJECT_TITLE=$(echo $PROJECT_INFO | jq -r '.project_title')
    PROJECT_ID=$(echo $PROJECT_INFO | jq -r '.project_id')
    PROJECT_UI=$(echo $PROJECT_INFO | jq -r '.project_ui')

    # Request and add Slack token & channel variables
    read -p "* Enter the client's Slack URL: "  slack_url_input
    addProjectVariable --id="$1" --key="SLACK_URL" --value="$slack_url_input" --sensitive=true

    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    RESPONSE=$(curl -s -X POST \
        -H "$HEADER" -H "$AUTH" \
        -d '{
            "type": "script",
            "script": "./slack.js",
            "states": [
                "complete"
            ],
            "events": [
                "environment.source-operation"
            ]
        }' \
        https://api.platform.sh/projects/$PROJECT_ID/integrations)
    # echo $RESPONSE | jq
}


operation() {
    PROJECT_INFO=$(getProjectInfoFromSubscription $1)
    PROJECT_TITLE=$(echo $PROJECT_INFO | jq -r '.project_title')
    PROJECT_ID=$(echo $PROJECT_INFO | jq -r '.project_id')
    PROJECT_UI=$(echo $PROJECT_INFO | jq -r '.project_ui')

    AUTH="Authorization: Bearer $(refreshOAuthToken)"
    RESPONSE=$(curl -X POST \
        -H "$HEADER" -H "$AUTH" \
        -d '{
            "operation": "npm-update"
        }' \
        https://api.platform.sh/projects/$PROJECT_ID/environments/$FLEET_UPDATES_BRANCH/source_operation)
    echo $RESPONSE
}
# runSourceOperation(){

# }

# merge() {

# }

# sync() {

# }

# backup () {

# }





"$@"

# args() {
#     for ARGUMENT in "$@"
#     do
#         KEY=$(echo $ARGUMENT | cut -f1 -d=)
#         VALUE=$(echo $ARGUMENT | cut -f2 -d=)   

#         case "$KEY" in
#                 --id)       ID=${VALUE} ;;
#                 --key)      ENVVAR_KEY=${VALUE} ;;
#                 --value)    ENVVAR_VALUE=${VALUE} ;;     
#                 *)   
#         esac    
#     done

#     echo "ID = $ID"
#     echo "key = $ENVVAR_KEY"
#     echo "value = $ENVVAR_VALUE"   
# }


# cat test.json | jq -r 'tos_entries[] | select(.value.project_title | contains("directus-demo")) | [ .value.project_id, .value.project_title, .value.project_ui ] | join("\t") '

            # "is_json": false,
            # "is_sensitive": false,
            # "visible_build": true,
            # "visible_runtime": true
            # "name": "'"$FLEET_PROFILE"'",
            # "value": "'"$FLEET_REPO"'"


# getVars() {
#     PROJECT_INFO=$(getProjectInfoFromSubscription $1)
#     PROJECT_TITLE=$(echo $PROJECT_INFO | jq -r '.project_title')
#     PROJECT_ID=$(echo $PROJECT_INFO | jq -r '.project_id')
#     PROJECT_UI=$(echo $PROJECT_INFO | jq -r '.project_ui')
#     # Add the variable.
#     AUTH="Authorization: Bearer $(refreshOAuthToken)"
#     RESPONSE=$(curl -s -X GET \
#         -H "$HEADER" -H "$AUTH" \
#         https://api.platform.sh/projects/$PROJECT_ID/variables)
#     echo $RESPONSE | jq
# }
