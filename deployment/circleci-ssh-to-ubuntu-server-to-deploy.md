# CircleCI ssh to Ubuntu server to deploy

1. [https://circleci.com/docs/2.0/add-ssh-key/](https://circleci.com/docs/2.0/add-ssh-key/)
2. Add public key to **authorized\_keys** by: **cat .ssh/id\_ed25519.pub > .ssh/authorized\_keys** (rename id\_ed25519.pub by your public key)
3. Add a run step

```
      - run:
          name: "Prepare Environment"
          command: |
            if [ -z `ssh-keygen -F '1.1.1.1'` ]; then
              ssh-keyscan -H '1.1.1.1' >> ~/.ssh/known_hosts
            fi
            ssh root@1.1.1.1 'cd src && ./autodeploy.sh'
```

Rename 1.1.1.1 by your ip server

