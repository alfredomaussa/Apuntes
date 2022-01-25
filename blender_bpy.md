Para instalar la blender api usar: `pip install bpy` pero ten cuidado, solo funciona con versión python3.7.

puedes obtener el último objeto creado con indexación 0 (blender indexa al reves)
``` python 
bpy.ops.mesh.primitive_cube_add()
cube = bpy.context.selected_objects[0]
```
``` python 
cube.dimensions=[tileSize,tileSize,tileExtrude]
cube.location=[0,0,-tileExtrude/2]
```

Cambiar el centro de referencia:
``` python 
bpy.ops.object.origin_set(type='ORIGIN_GEOMETRY', center='BOUNDS')
```

[acá](https://blender.stackexchange.com/a/38626/65865) dice como seleccionar objetos:
``` python
bpy.data.objects["Cube"].select_set(True)
# to select the object in the 3D viewport,

current_state = bpy.data.objects["Cube"].select_get()
# retrieving the current state

# this way you can also select multiple objects

bpy.context.view_layer.objects.active = bpy.data.objects['Sphere']
# to set the active object
```
