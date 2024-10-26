### The following instrucions are based on an nx monoremo (pnpm workspace) but it should generally be applicable for a standalone nextjs project as well.

> #### This script does the follwing :
>
> - Build the project in .temp dir
> - Backup previous build dir (.next)
> - Swap the old build dir (.next) with the new buid dir (.temp)
> - Start the project with pm2 (restart if it's an existing project)

`deploy.sh` Deploy script - should be placed within the root dir besides next.config.js file.

```bash
#!/bin/bash

# Any unique project name on current machine. It can be different from the name in package.json
NAME='any-unique-name'
BACKUP_DIR="${HOME}/deployment-backups"

# ----------------------------- do not modify anything below -----------------------------

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

# Check if package.json exists
if [ ! -f package.json ]; then
  echo -e "${RED}package.json not found!${NC}"
  exit 1
fi

# Extract the project name using jq
NAME_IN_PACKAGE_JSON=$(jq -r '.name' package.json)
echo -e "${GREEN}Project name: ${NAME_IN_PACKAGE_JSON}${NC}"

# Check if jq command was successful
if [ $? -ne 0 ]; then
  echo "${RED}Failed to parse package.json${NC}"
  exit 1
fi

START_TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")

echo "Starting deployment for ${NAME}..."

echo "Pulling latest changes..."
git pull

echo "Installing dependencies..."
pnpm i

echo "Building..."
export NEXTJS_BUILD_DIR=.temp
export NX_DAEMON=false
npx nx build $NAME_IN_PACKAGE_JSON --verbose

if [ $? -eq 0 ]; then
  echo -e "${GREEN}Build successful!${NC}"
else
  echo -e "${RED}Build failed!${NC}"
  rm -rf .temp
  exit 1
fi

export NEXTJS_BUILD_DIR=.next

if [ ! -d ".temp" ]; then
  echo -e "${RED}\".temp\" dir does not exist!${NC}"
  exit 1
fi

if [ -d ".next" ]; then
  PROJECT_BACKUP_DIR="${BACKUP_DIR}/${NAME}_${START_TIMESTAMP}"
  echo "Backing up previous deployment to \"${PROJECT_BACKUP_DIR}\""
  mkdir -p $PROJECT_BACKUP_DIR
  mv .next "${PROJECT_BACKUP_DIR}/.next"
fi

mv .temp .next

## check if app exists in pm2
if pm2 describe $NAME &>/dev/null; then
  echo "${NAME} is running, reloading..."
  pm2 reload $NAME --update-env
else
  echo "${NAME} is not running, starting..."
  pm2 start npm --name $NAME --time -- start
fi

pm2 save
echo -e "${GREEN}Deployment completed successfully!${NC}"
```

### Add the following line to the `next.config.js` file

```diff
/** @type {import("next").NextConfig} */
const config = {
+ distDir: process.env.NEXTJS_BUILD_DIR ?? '.next',
  reactStrictMode: true,
  i18n: {
    locales: ["en", "sv"],
    defaultLocale: "sv",
  },
  transpilePackages: ["geist"],
};

module.exports = config;
```
