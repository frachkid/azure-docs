---
title: Continuous integration and continuous deployment - Azure IoT Edge | Microsoft Docs
description: Set up continuous integration and continuous deployment - Azure IoT Edge with Azure DevOps, Azure Pipelines
author: shizn
manager: philmea
ms.author: xshi
ms.date: 08/20/2019
ms.topic: conceptual
ms.service: iot-edge
services: iot-edge
ms.custom: seodec18
---

# Continuous integration and continuous deployment to Azure IoT Edge

You can easily adopt DevOps with your Azure IoT Edge applications with the built-in Azure IoT Edge tasks in Azure Pipelines. This article demonstrates how you can use the continuous integration and continuous deployment features of Azure Pipelines to build, test, and deploy applications quickly and efficiently to your Azure IoT Edge. 

![Diagram - CI and CD branches for development and production](./media/how-to-ci-cd/cd.png)

In this article, you learn how to use the built-in Azure IoT Edge tasks for Azure Pipelines to create two pipelines for your IoT Edge solution. There are four actions can be used in the Azure IoT Edge tasks.
   - **Azure IoT Edge - Build Module images** takes your IoT Edge solution code and builds the container images.
   - **Azure IoT Edge - Push Module images** pushes module images to the container registry you specified.
   - **Azure IoT Edge - Generate Deployment Manifest** takes a deployment.template.json file and the variables, then generates the final IoT Edge deployment manifest file.
   - **Azure IoT Edge - Deploy to IoT Edge devices** helps create IoT Edge deployments to single/multiple IoT Edge devices.

## Prerequisites

* An Azure Repos repository. If you don't have one, you can [Create a new Git repo in your project](https://docs.microsoft.com/azure/devops/repos/git/create-new-repo?view=vsts&tabs=new-nav).
* An IoT Edge solution committed and pushed to your repository. If you want to create a new sample solution for testing this article, follow the steps in [Develop and debug modules in Visual Studio Code](how-to-vs-code-develop-module.md) or [Develop and debug C# modules in Visual Studio](how-to-visual-studio-develop-csharp-module.md).
   * For this article, all you need is the solution folder created by the IoT Edge templates in either Visual Studio Code or Visual Studio. You don't need to build, push, deploy, or debug this code before proceeding. You'll set those processes up in Azure Pipelines. 
   * If you're creating a new solution, clone your repository locally first. Then, when you create the solution you can choose to create it directly in the repository folder. You can easily commit and push the new files from there. 
* A container registry where you can push module images. You can use [Azure Container Registry](https://docs.microsoft.com/azure/container-registry/) or a third-party registry. 
* An active [IoT hub](../iot-hub/iot-hub-create-through-portal.md) with at least IoT Edge devices for testing the separate test and production deployment stages. You can follow the quickstart articles to create an IoT Edge device on [Linux](quickstart-linux.md) or [Windows](quickstart.md)


For more information about using Azure Repos, see [Share your code with Visual Studio and Azure Repos](https://docs.microsoft.com/azure/devops/repos/git/share-your-code-in-git-vs?view=vsts)

## Configure continuous integration
In this section, you create a new build pipeline. Configure the pipeline to run automatically when you check in any changes to the sample IoT Edge solution, and publish build logs.

>[!NOTE]
>This article uses the Azure DevOps visual designer. Before you follow the steps in this section, turn off the preview feature for the new YAML pipeline creation experience. 
>1. In Azure DevOps, select your profile icon then select **Preview features**.
>2. Turn **New YAML pipeline creation experience** off. 
>
>For more information, see [Create a build pipeline](https://docs.microsoft.com/azure/devops/pipelines/get-started-designer?view=vsts&tabs=new-nav#create-a-build-pipeline).

1. Sign into your Azure DevOps organization (**https:\//dev.azure.com/{your organization}/**) and open the project that contains your IoT Edge solution repository.

   For this article, we created a repository called **IoTEdgeRepo**. That repository contains **IoTEdgeSolution** which has the code for a module named **filtermodule**. 

   ![Open your DevOps project](./media/how-to-ci-cd/init-project.png)

2. Navigate to Azure Pipelines in your project. Open the **Builds** tab and select **New pipeline**. Or, if you already have build pipelines, select the **New** button. Then choose **New build pipeline**.

    ![Create a new build pipeline](./media/how-to-ci-cd/add-new-build.png)

3. Follow the prompts to create your pipeline. 

   1. Provide the source information for your new build pipeline. Select **Azure Repos Git** as the source, then select the project, repository, and branch where your IoT Edge solution code is located. Then, select **Continue**. 

      ![Select your pipeline source](./media/how-to-ci-cd/pipeline-source.png)

   2. Select **Empty job** instead of a template. 

      ![Start with an empty process](./media/how-to-ci-cd/start-with-empty.png)

4. Once your pipeline is created, you are taken to the pipeline editor. In your pipeline description, choose the correct agent pool based on your target platform: 
    
   * If you would like to build your modules in platform amd64 for Linux containers, choose **Hosted Ubuntu 1604**

   * If you would like to build your modules in platform amd64 for Windows 1809 containers, you need to [set up self-hosted agent on Windows](https://docs.microsoft.com/azure/devops/pipelines/agents/v2-windows?view=vsts).

   * If you would like to build your modules in platform arm32v7 or arm64 for Linux containers, you need to [set up self-hosted agent on Linux](https://blogs.msdn.microsoft.com/iotdev/2018/11/13/setup-azure-iot-edge-ci-cd-pipeline-with-arm-agent/).
    
     ![Configure build agent pool](./media/how-to-ci-cd/configure-env.png)

5. Your pipeline comes preconfigured with a job called **Agent job 1**. Select the plus sign (**+**) to add three tasks to the job: **Azure IoT Edge** twice, **Copy Files** once and **Publish Build Artifacts** once. (Hover over the name of each task to see the **Add** button.)

   ![Add Azure IoT Edge task](./media/how-to-ci-cd/add-iot-edge-task.png)

   When all four tasks are added, your Agent job looks like the following example:
    
   ![Three tasks in the build pipeline](./media/how-to-ci-cd/add-tasks.png)

6. Select the first **Azure IoT Edge** task to edit it. This task builds all modules in the solution with the target platform that you specify.

   * **Display name**: Accept the default **Azure IoT Edge - Build module images**.
   * **Action**: Accept the default **Build module images**. 
   * **.template.json file**: Select the ellipsis (**...**) and navigate to the **deployment.template.json** file in the repository that contains your IoT Edge solution. 
   * **Default platform**: Select the appropriate platform for your modules based on your target IoT Edge device. 
   * **Output variables**: The output variables include a reference name that you can use to configure the file path where your deployment.json file will be generated. Set the reference name to something memorable like **edge**. 

7. Select the second **Azure IoT Edge** task to edit it. This task pushes all module images to the container registry that you select.

   * **Display name**: The display name is automatically updated when the action field changes. 
   * **Action**: Use the dropdown list to select **Push module images**. 
   * **Container registry type**: Select the type of container registry that you use to store your module images. Depending on which registry type you choose, the form changes. If you choose **Azure Container Registry**, use the dropdown lists to select the Azure subscription and the name of your container registry. If you choose **Generic Container Registry**, select **New** to create a registry service connection. 
   * **.template.json file**: Select the ellipsis (**...**) and navigate to the **deployment.template.json** file in the repository that contains your IoT Edge solution. 
   * **Default platform**: Select the same platform as your built module images.

   If you have multiple container registries to host your module images, you need to duplicate this task, select different container registry, and use **Bypass module(s)** in the advanced settings to bypass the images which are not for this specific registry.

8. Select the **Copy Files** task to edit it. Use this task to copy files to the artifact staging directory.

   * **Display name**: Copy Files to: Drop folder.
   * **Contents**: Put two lines in this section, `deployment.template.json` and `**/module.json`. These two types of files are the inputs to generate IoT Edge deployment manifest. Need to be copied to the artifact staging folder and published for release pipeline.
   * **Target Folder**: Put the variable `$(Build.ArtifactStagingDirectory)`. See [Build variables](https://docs.microsoft.com/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml#build-variables) to learn about the description.

9. Select the **Publish Build Artifacts** task to edit it. Provide artifact staging directory path to the task so that the path can be published to release pipeline.
   
   * **Display name**: Publish Artifact: drop.
   * **Path to publish**: Put the variable `$(Build.ArtifactStagingDirectory)`. See [Build variables](https://docs.microsoft.com/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml#build-variables) to learn about the description.
   * **Artifact name**: drop.
   * **Artifact publish location**: Azure Pipelines.


10. Open the **Triggers** tab and check the box to **Enable continuous integration**. Make sure the branch containing your code is included.

    ![Turn on continuous integration trigger](./media/how-to-ci-cd/configure-trigger.png)

11. Save the new build pipeline with **Save** button.

This pipeline is now configured to run automatically when you push new code to your repo. The last task, publishing the pipeline artifacts, triggers a release pipeline. Continue to the next section to build the release pipeline. 

## Configure continuous deployment
In this section, you create a release pipeline that is configured to run automatically when your build pipeline drops artifacts, and it will show deployment logs in Azure Pipelines.

Create a new pipeline, and add a new stage 

1. In the **Releases** tab, choose **+ New pipeline**. Or, if you already have release pipelines, choose the **+ New** button and select **+ New release pipeline**.  

    ![Add release pipeline](./media/how-to-ci-cd/add-release-pipeline.png)

2. When prompted to select a template, choose to start with an **Empty job**.

    ![Start with an empty job](./media/how-to-ci-cd/start-with-empty-job.png)

3. Your new release pipeline initializes with one stage, called **Stage 1**. Rename Stage 1 to **dev** and treat it as a test environment. Usually, continuous deployment pipelines have multiple stages including **dev**, **staging** and **prod**. You can create more based on your DevOps practice. Close the stage details window once it's renamed. 

4. Link the release to the build artifacts that are published by the build pipeline. Click **Add** in artifacts area.

   ![Add artifacts](./media/how-to-ci-cd/add-artifacts.png)  
    
5. In **Add an artifact page**, select source type **Build**. Then, select the project and the build pipeline you created. Then, select **Add**.

   ![Add a build artifact](./media/how-to-ci-cd/add-an-artifact.png)

6. Open the artifact triggers and select the toggle to enable the continuous deployment trigger. Now, a new release will be created each time a new build is available.

   ![Configure continuous deployment trigger](./media/how-to-ci-cd/add-a-trigger.png)

7. The **dev** stage is preconfigured with one job and zero tasks. From the pipeline menu, select **Tasks** then choose the **dev** stage.  Select the job and task count to configure the tasks in this stage.

    ![Configure dev tasks](./media/how-to-ci-cd/view-stage-tasks.png)

8. In the **dev** stage, you should see a default **Agent job**. You can configure details about the agent job, but the deployment task is platform insensitive so you can use either **Hosted VS2017** or **Hosted Ubuntu 1604** in the **Agent pool** (or any other agent managed by yourself). 

9. Select the plus sign (**+**) to add two task. Search for and add **Azure IoT Edge** twice.

    ![Add tasks for dev](./media/how-to-ci-cd/add-task-qa.png)

10. Select the first **Azure IoT Edge** task and configure it with the following values:

    * **Display name**: The display name is automatically updated when the action field changes. 
    * **Action**: Use the dropdown list to select **Generate deployment manifest**. Changing the action value also updates the task display name to match.
    * **.template.json file**: Put the path `$(System.DefaultWorkingDirectory)/Drop/drop/deployment.template.json`. The path is published from build pipeline.
    * **Default platform**: Choose the same value when building the module images.
    * **Output path**: Put the path `$(System.DefaultWorkingDirectory)/Drop/drop/configs/deployment.json`. This path is the final IoT Edge deployment manifest file.

    These configurations helps replace the module image URLs in the `deployment.template.json` file. The **Generate deployment manifest** also helps replace the variables with the exact value you defined in the `deployment.template.json` file. In VS/VS Code, you are specifying the actual value in a `.env` file. In Azure Pipelines, you set the value in Release Pipeline Variables tab. Move to Variables tab and configure the Name and Value as following.

    * **ACR_ADDRESS**: Your Azure Container Registry address. 
    * **ACR_PASSWORD**: Your Azure Container Registry password.
    * **ACR_USER**: Your Azure Container Registry username.

    If you have other variables in your project, you can specify the name and value in this tab. The **Generate deployment manifest** can only recognize the variables are in `${VARIABLE}` flavor, make sure you are using this in your `*.template.json` files.

    ![Configure variables for release pipeline](./media/how-to-ci-cd/configure-variables.png)

10. Select the second **Azure IoT Edge** task and configure it with the following values:

    * **Display name**: The display name is automatically updated when the action field changes. 
    * **Action**: Use the dropdown list to select **Deploy to IoT Edge devices**. Changing the action value also updates the task display name to match.
    * **Azure subscription**: Select the subscription that contains your IoT Hub.
    * **IoT Hub name**: Select your IoT hub. 
    * **Choose single/multiple device**: Choose whether you want the release pipeline to deploy to one device or multiple devices. 
      * If you deploy to a single device, enter the **IoT Edge device ID**. 
      * If you are deploying to multiple devices, specify the device **target condition**. The target condition is a filter to match a set of IoT Edge devices in IoT Hub. If you want to use Device Tags as the condition, you need to update your corresponding devices Tags with IoT Hub device twin. Update the **IoT Edge deployment ID** and **IoT Edge deployment priority** in the advanced settings. For more information about creating a deployment for multiple devices, see [Understand IoT Edge automatic deployments](module-deployment-monitoring.md).
    * Expand Advanced Settings, select **IoT Edge deployment ID**, put the variable `$(System.TeamProject)-$(Release.EnvironmentName)`. This maps the project and release name with your IoT Edge deployment ID.

11. Select **Save** to save your changes to the new release pipeline. Return to the pipeline view by selecting **Pipeline** from the menu. 
    
## Verify IoT Edge CI/CD with the build and release pipelines

To trigger a build job, you can either push a commit to source code repository or manually trigger it. In this section, you manually trigger the CI/CD pipeline to test that it works. Then verify that the deployment succeeds.

1. Navigate to the build pipeline that you created at the beginning of this article. 

2. You can trigger a build job in your build pipeline by selecting the **Queue** button as in following screenshot.

    ![Manual trigger](./media/how-to-ci-cd/manual-trigger.png)

3. Select the build job to watch its progress. If the build pipeline is completed successfully, it triggers a release to **dev** stage. 

    ![Build logs](./media/how-to-ci-cd/build-logs.png)

4. The successful **dev** release creates IoT Edge deployment to target IoT Edge devices.

    ![Release to dev](./media/how-to-ci-cd/pending-approval.png)

5. Click **dev** stage to see release logs.

    ![Release logs](./media/how-to-ci-cd/release-logs.png)



## Next steps
* IoT Edge DevOps best practices sample in [Azure DevOps Project for IoT Edge](how-to-devops-project.md)
* Understand the IoT Edge deployment in [Understand IoT Edge deployments for single devices or at scale](module-deployment-monitoring.md)
* Walk through the steps to create, update, or delete a deployment in [Deploy and monitor IoT Edge modules at scale](how-to-deploy-monitor.md).
