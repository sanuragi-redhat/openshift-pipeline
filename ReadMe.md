   GitHub repo
      ↓
   Tekton Pipeline
      ↓
   Build image (buildah)
      ↓
   Push → OpenShift Internal Registry
      ↓
   Deploy (oc apply)


create tekton pipeline

      oc project pipeline
      
      oc policy add-role-to-user edit system:serviceaccount:pipeline:pipeline
      oc policy add-role-to-user system:image-builder system:serviceaccount:pipeline:pipeline
      oc adm policy add-scc-to-user privileged -z pipeline -n pipeline
   
      
      [lab-user@bastion app]$ oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
      task.tekton.dev/git-clone configured
      
      [lab-user@bastion app]$ oc get task | grep git-clone
      git-clone     2m43s
      
      [lab-user@bastion app]$ cat imagestream.yaml
      
      apiVersion: image.openshift.io/v1
      kind: ImageStream
      metadata:
        name: simpleservice
        namespace: pipeline
      
      oc apply -f imagestream.yaml
      
      
      [lab-user@bastion app]$ cat deploy-task.yaml
      apiVersion: tekton.dev/v1
      kind: Task
      metadata:
        name: deploy-app
        namespace: pipeline
      spec:
        workspaces:
          - name: source
        steps:
          - name: deploy
            image: quay.io/openshift/origin-cli:latest
            script: |
              #!/bin/sh
              set -e
      
              cd $(workspaces.source.path)/app
              oc apply -k .
      
      
      [lab-user@bastion app]$ cat pipeline.yaml
      apiVersion: tekton.dev/v1
      kind: Pipeline
      metadata:
        name: build-deploy-pipeline
        namespace: pipeline
      spec:
        workspaces:
          - name: shared-workspace
      
        params:
          - name: IMAGE
            default: image-registry.openshift-image-registry.svc:5000/pipeline/simpleservice:latest
      
        tasks:
      
          # ✅ Clone (simple reference)
          - name: clone
            taskRef:
              name: git-clone
            workspaces:
              - name: output
                workspace: shared-workspace
            params:
              - name: url
                value: https://github.com/sanuragi-redhat/gitops-repo.git
              - name: revision
                value: main
      
          # ✅ Build
          - name: build
            runAfter: [clone]
            taskRef:
              name: build-image
            workspaces:
              - name: source
                workspace: shared-workspace
            params:
              - name: IMAGE
                value: $(params.IMAGE)
      
          # ✅ Deploy
          - name: deploy
            runAfter: [build]
            taskRef:
              name: deploy-app
            workspaces:
              - name: source
                workspace: shared-workspace
      
      
      [lab-user@bastion app]$ cat pipelinerun.yaml
      apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: build-deploy-run-
        namespace: pipeline
      spec:
        serviceAccountName: pipeline
      
        pipelineRef:
          name: build-deploy-pipeline
      
        workspaces:
          - name: shared-workspace
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 2Gi
      
      
      [lab-user@bastion app]$ oc get tasks
      NAME          AGE
      build-image   9m10s
      deploy-app    8m50s
      git-clone     6m47s
      
      
