[Java Scene Builder讲解](https://blog.csdn.net/cnds123321/article/details/104507487?utm_medium=distribute.pc_relevant.none-task-blog-searchFromBaidu-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-searchFromBaidu-2.control)

###### 1.添加 archetype 

setting->Maven->Create from ArcheType->Add ArcheType

~~~
弹框填写：
the groupId (org.openjfx),
the artifactId (javafx-maven-archetypes)
the version (0.0.1)
~~~

###### 2.使用ArcheType创建项目

~~~
修改ArcheTypeArtifactid的值为：
javafx-archetype-fxml， javafx 项目包含 fxml
javafx-archetype-simple， javafx 项目不包含 fxml
~~~

###### 3.JavaFX之多个FXML加载和通信

#### Main.java

```java
import javafx.application.Application;
import javafx.fxml.FXMLLoader;
import javafx.scene.Parent;
import javafx.scene.Scene;
import javafx.stage.Stage;

public class Main extends Application {

    @Override
    public void start(Stage primaryStage) throws Exception{
        Parent root = FXMLLoader.load(getClass().getResource("main.fxml"));
        primaryStage.setTitle("JFeonix Test Example");
        primaryStage.setScene(new Scene(root, 600, 400));
        primaryStage.show();
    }


    public static void main(String[] args) {
        launch(args);
    }
}
```

#### Controller

##### MainController.java

```java
package controller;

import com.jfoenix.controls.JFXTextArea;
import javafx.fxml.FXML;

public class MainController {
    //一定要初始化这两个
    //并且，这个变量名一定是这样的模式：`<fx:id>Controller`
    //Eg，你的example.fxml中的`fx:id="example"`，那么你的变量名一定是`exampleController`
    //如果不信，欢迎尝试`NPE` :D
    //PS:fxml或者java文件名无所谓，只看`MainController`变量名，肉测!
    @FXML
    private Part1Controller part1Controller;

    @FXML
    private Part2Controller part2Controller;


    @FXML
    private void initialize() {
        part1Controller.injectMainController(this);
    }

    JFXTextArea getOutputPane() {
       return part2Controller.textArea;
    }
}
```

##### Part1Controller.java

```java
package controller;

import com.jfoenix.controls.JFXTextArea;
import javafx.fxml.FXML;
import javafx.scene.control.Button;

public class Part1Controller {

    @FXML
    private Button button;

    private MainController mainController;
    //注入MainController
    void injectMainController(MainController mainController) {
        this.mainController = mainController;
    }

    //得到想要的TextArea
    private JFXTextArea getTextArea () {
        return this.mainController.getOutputPane();
    }

    @FXML
    private void onButtonAction() {
        getTextArea().appendText("Fuck me\n");
    }

    @FXML
    private void initialize() {
    }
}
```

##### Part2Controller.java

```java
package controller;

import com.jfoenix.controls.JFXTextArea;
import javafx.fxml.FXML;

public class Part2Controller {
    @FXML
    public JFXTextArea textArea;

    @FXML
    private void initialize() {

    }
}
```

### resources

#### main.fxml

```fxml
<?xml version="1.0" encoding="UTF-8"?>


<?import javafx.scene.layout.AnchorPane?>
<?import javafx.scene.layout.HBox?>
<AnchorPane prefHeight="400.0" prefWidth="600.0" xmlns="http://javafx.com/javafx/9" xmlns:fx="http://javafx.com/fxml/1"
            fx:controller="controller.MainController">
    <children>
        <HBox layoutX="163.0" layoutY="135.0" prefHeight="100.0" prefWidth="200.0">
            <children>
                <fx:include fx:id="part1" source="part1.fxml"/>
                <fx:include fx:id="part2" source="part2.fxml"/>
            </children>
        </HBox>
    </children>
</AnchorPane>
```

#### part1.fxml

```fxml
<?xml version="1.0" encoding="UTF-8"?>


<?import javafx.scene.control.Button?>
<?import javafx.scene.layout.StackPane?>
<StackPane maxHeight="-Infinity" maxWidth="-Infinity" minHeight="-Infinity" minWidth="-Infinity" prefHeight="174.0"
           prefWidth="191.0" xmlns="http://javafx.com/javafx/9" xmlns:fx="http://javafx.com/fxml/1"
           fx:controller="controller.Part1Controller">
    <children>
        <Button fx:id="button" mnemonicParsing="false" onAction="#onButtonAction" text="Button"/>
    </children>
</StackPane>
```

#### part2.fxml

```fxml
<?xml version="1.0" encoding="UTF-8"?>


<?import com.jfoenix.controls.JFXTextArea?>
<?import javafx.scene.layout.StackPane?>
<StackPane maxHeight="-Infinity" maxWidth="-Infinity" minHeight="-Infinity"
           minWidth="-Infinity" prefHeight="192.0" prefWidth="289.0" xmlns="http://javafx.com/javafx/9"
           xmlns:fx="http://javafx.com/fxml/1" fx:controller="controller.Part2Controller">
    <children>
        <JFXTextArea fx:id="textArea"/>
    </children>
</StackPane>
```