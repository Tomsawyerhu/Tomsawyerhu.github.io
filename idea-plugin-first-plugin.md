---
layout: page
title: IDEA-PLUGIN-WITH-GRADLE-GET-STARTED
ftitle: IDEA Plugin Development
---

## Creating a Gradle-Based IntelliJ Platform Plugin with New Project Wizard
You need to click the following buttons in a sequential order:

>File `|` New `|` Project

### New Project Configuration Screen

On the first screen, the type of project is configured:

+ From the project type pane on the left, choose Gradle.

+ Specify the Project SDK based on the Java 8 JDK.

  - When targeting 2020.3 and later only, using Java 11 is now required.

+ In the Additional Libraries and Frameworks panel, select Java and IntelliJ Platform Plugin.
+ Optionally, you can add kotlin support.
+ Click *Next*

![](/public/pic/idea_plugin/1.png)

### Project Structure
```
my_gradle_plugin
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
└── src
├── main
│   ├── java
│   └── resources
│       └── META-INF
│           └── plugin.xml
└── test
├── java
└── resource
```

### Running a Simple Gradle-Based IntelliJ Platform Plugin
Gradle projects are run from the IDE's Gradle Tool window.

#### Adding Code to the Project

**Step1** Creating a Custom Action

We need to extend the given abstract class *AnAction*.

```
public class PopupDialogAction extends AnAction {

  @Override
  public void update(AnActionEvent e) {
    // Using the event, evaluate the context, and enable or disable the action.
  }

  @Override
  public void actionPerformed(@NotNull AnActionEvent e) {
    // Using the event, implement an action. For example, create and show a dialog.
  }
}
```

**Step2** Registering a Custom Action

Add the following text to [build.gradle](#Project Structure).

```
<actions>
  <action id="org.intellij.sdk.action.PopupDialogAction" class="org.intellij.sdk.action.PopupDialogAction"
        text="Pop Dialog Action" description="SDK action example">
    <add-to-group group-id="ToolsMenu" anchor="first"/>
  </action>
</actions>
```

You can also realize this by keyboard shortcut  *Alt + Shift + Enter*.

**Step3** Extending Method Body in Step 1

```
@Override
public void actionPerformed(@NotNull AnActionEvent event) {
  // Using the event, create and show a dialog
  Project currentProject = event.getProject();
  StringBuffer dlgMsg = new StringBuffer(event.getPresentation().getText() + " Selected!");
  String dlgTitle = event.getPresentation().getDescription();
  // If an element is selected in the editor, add info about it.
  Navigatable nav = event.getData(CommonDataKeys.NAVIGATABLE);
  if (nav != null) {
    dlgMsg.append(String.format("\nSelected Element: %s", nav.toString()));
  }
  Messages.showMessageDialog(currentProject, dlgMsg.toString(), dlgTitle, Messages.getInformationIcon());
}

@Override
public void update(AnActionEvent e) {
  // Set the availability based on whether a project is open
  Project project = e.getProject();
  e.getPresentation().setEnabledAndVisible(project != null);
}
```

#### Executing the Plugin

* Open the Gradle tool window and search for the **runIde** task.
![](/public/pic/idea_plugin/2.png)

* Double click to start sandbox.

#### Result

* Find *Pop Dialog Action* in *Tools*.
![](/public/pic/idea_plugin/3.png)

* Select a file or a directory in the project, then click *Pop Dialog Action*.
![](/public/pic/idea_plugin/4.png)
