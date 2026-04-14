i have basic nestjs app

go to aws and create identity provider for your app

Identity provider: Github

crete a role with web identity provider and attach the policy with permissions to access AWS resources

choose github org and repo and branch as optional

git remote set-url origin git@github.com-personal/rashidpulikkuth/nestjs-aws-app.git
git push origin main


create a workflow file in .github/workflows/

its ci/cd so best name

.github/workflows/