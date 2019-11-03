UnityHack
====
A set of snippets to extend Unity Editor.

Before you start
----
All of snippets from this repository only work inside Unity Editor.<br>
You must place your code inside `Editor` directory to avoid build errors.

Bootstrap your plugin
----
It all begins with bootstrapping code. This step is pretty easy because Unity already has API for this.
```cs
[InitializeOnLoadMethod]
public static void Initialize() {
    /* ... Your code goes here ... */
}
```

Input
----
Because of `Input` class does not work on Unity Editor, you have to write special code to handle it.

__Handling global editor input__
```cs
System.Reflection.FieldInfo info = typeof(EditorApplication).GetField("globalEventHandler", System.Reflection.BindingFlags.Static | System.Reflection.BindingFlags.NonPublic);
EditorApplication.CallbackFunction value = (EditorApplication.CallbackFunction)info.GetValue(null);
value += HandleUnityEvent;
info.SetValue(null, value);
```
```cs
static void HandleUnityEvent()
{
    var e = Event.current;

    if (e.type == EventType.KeyDown && e.keyCode == KeyCode.LeftControl)
    {
        /* Do your stuffs here */
    }
}
```

__Handling scene window input__<br>
If you want to execute your code __ONLY__ inside the scene view, try this instead of `globalEventHandler`. It also seems less scary.
```cs
SceneView.duringSceneGui += HandleSceneEvent;
```
```cs
static void HandleSceneEvent() {
    var e = Event.current;
                
    if (e == null) return;
    if (e.type == EventType.KeyDown && e.keyCode == KeyCode.LeftControl)
    {
        /* Do your stuffs here */
    }
}
```


Property change detection
----
There's no official API to get notification when your changed the property. However, this can be done with `Undo.postprocessModifications` since every changes will be recorded to `Undo`.

```cs
Undo.postprocessModifications += OnPostProcessModifications;
```
```cs
private static UndoPropertyModification[] OnPostProcessModifications(UndoPropertyModification[] propertyModifications)
{
    foreach (var m in propertyModifications)
    {
        if (m.currentValue.target is Text textTarget) {
            if (m.currentValue.propertyPath == "m_FontData.m_FontSize") {
                /* User has changed `FontSize` from `Text` Component. */
            }
        }
    }
}
```

Graphics inside SceneView
----
`OnDrawGizmo` is a traditional way to draw something inside SceneView. However, you must have a `GameObject` to receive `OnDrawGizmo` callback from Unity.<br>
You can also use `SceneView.duringSceneGui` to draw your stuffs without active `GameObject`.

```cs
SceneView.duringSceneGui += OnDrawSceneGUI;
```
```cs
static void OnDrawSceneGUI()
{
    /* Do your stuffs! */
}
```

__Render Texts__<br>
There are APIs for drawing `Rect`, `Line`, `Polygon` and even `Texture`. But they don't have an API to render texts.<br>
You can render texts by below script:
```cs
public static void DrawString(string text, Vector3 position, Color? color = null, bool showTexture = true, int offsetY = 0)
{
    Handles.BeginGUI();

    var restoreColor = GUI.color;
    var viewSize = SceneView.currentDrawingSceneView.position;
    if (color.HasValue) GUI.color = color.Value;
    var view = SceneView.currentDrawingSceneView;
    Vector3 screenPos = view.camera.WorldToScreenPoint(
        position + view.camera.transform.forward);
    if (screenPos.y < 0 || screenPos.y > Screen.height || screenPos.x < 0 || screenPos.x > Screen.width || screenPos.z < 0)
    {
        GUI.color = restoreColor;
        Handles.EndGUI();
        return;
    }
    var style = new GUIStyle(GUI.skin.label);
    style.alignment = TextAnchor.MiddleCenter;
    style.fontSize = 30;
    Vector2 size = style.CalcSize(new GUIContent(text));
    GUI.Label(new Rect(screenPos.x - size.x/2, viewSize.height - screenPos.y - offsetY - size.y/2, size.x, size.y), text, style);
    GUI.color = restoreColor;

    Handles.EndGUI();
}
```

Object creation detection
----

__Detect GameObject creation__
```cs
EditorApplication.hierarchyChanged += OnHierarchyChanged;
```
```cs
private static GameObject lastChangedGO;

private static void OnHierarchyChanged()
{
    if (Selection.objects.Length == 0)
        return;
    var go = Selection.objects[0] as GameObject;
    if (go == null)
        return;

    if (go == lastChangedGO)
        return;

    lastChangedGO = go;

    /* Do your stuff! */ 
}
```

__Detect Component creation__
```cs
private static GameObject lastChangedGO;
private static int lastGOComponentCount;

private static void OnHierarchyChanged()
{
    if (Selection.objects.Length == 0)
        return;
    var go = Selection.objects[0] as GameObject;
    if (go == null)
        return;

    if (go != lastChangedGO)
        return;
    lastChangedGO = go;

    var comps = go.GetComponents<Component>();
    if (comps.Length == lastGOComponentCount)
        return;
    lastGOComponentCount = comps.Length;

    var addedComponent = comps.Last();
    /* Do your stuff! */ 
}
```


Drag and Drop
----

__Asset drag detection__<br>
You can watch `BeginDrag` for your assets with below code:
```cs
EditorApplication.update += OnUpdate;
```
```cs
static void OnUpdate()
{
    if (DragAndDrop.paths.Length > 0 &&
        !(Selection.activeObject is GameObject))
    {
        // DragAndDrop.objectReferences;
    }
}
```