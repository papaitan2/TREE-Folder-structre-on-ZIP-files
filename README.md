


# TREE-Folder-structre-on-ZIP-files



`tr` es una herramienta que te ayuda a visualizar la estructura de tus carpetas de una forma más clara e informativa. No solo te muestra las carpetas y archivos dentro de una ubicación, sino que también puede adentrarse en archivos comprimidos como .zip, .rar y .7z para mostrarte su contenido.

Puedes ver el "esqueleto" de tus carpetas y archivos, con colores que resaltan las extensiones y iconos que diferencian los tipos de elementos. Además, `tr` te permite ordenar todo por nombre, fecha o tamaño, para que encuentres lo que buscas más rápidamente.

Guarda esta vista organizada en un archivo HTML, como si fuera una foto de tu estructura de archivos, que puedes abrir en tu navegador.

En resumen, `tr` ofrece una forma más intuitiva de entender cómo están organizados tus archivos y carpetas, incluso dentro de archivos comprimidos.

## El código :
---
<br>
<br>
<br>



```python
'''
Mejoras e implementar :

- Que junto al nombre de carpeta salga su volumen de megas
- Que las carpetas se han escamoteables
- Hacer que al procesar el nombre sólo tenga en cuenta el último punto
para que no acabe todo el nombre del archivo en rojo si hay varios puntos



'''

from urllib.parse import quote
import pathlib
import os
from datetime import datetime
import zipfile
import rarfile
import py7zr

from rich import print
from rich.filesize import decimal
from rich.markup import escape
from rich.text import Text
from rich.tree import Tree
from rich.console import Console


def obtener_ruta_completa(ruta, nombre_archivo):
  """
  Obtiene la ruta completa de un archivo a partir de una ruta y un nombre de archivo.
  Args:
    ruta: La ruta al directorio donde se encuentra el archivo.
    nombre_archivo: El nombre del archivo.
  Returns:
    La ruta completa del archivo.
  """

  ruta_completa = os.path.join(ruta, nombre_archivo)
  return ruta_completa

def get_icon(path):
    """Determine the icon based on file suffix."""
    icons = {
        ".py": "🐍 ",
        ".ipynb": "🪐 ",
        ".png": "📷 ",
        ".jpg": "📷 ",
        ".jpeg": "📷 ",
        ".gif": "📷 ",
        ".svg": "📷 ",
        ".txt": "🚲 ",
        ".doc": "🛺 ",
        ".md": "🚲 ",
        ".pdf": "📕 ",
        ".epub": "📕 ",
        ".docx": "🗃️ ",
        ".sql": "🗄️ ",
        ".db": "🗄️ ",
        ".zip": "📦 ",
        ".rar": "📦 ",
        ".7z": "📦 ",
        ".exe": "⚙️ ",
        ".msi": "⚙️ ",
        ".htm": "🌐 ",
        ".html": "🌐 ",
        ".mp4": "📺 ",
        ".avi": "📺 ",
        ".mov": "📺 ",
        ".mkv": "📺 ",
        ".webm": "📺 ",
        ".mp3": "🎸 ",
        ".wav": "🎸 ",
        ".aac": "🎸 ",
        ".flac": "🎸 ",
        ".ogg": "🎸 ",
        ".wma": "🎸 ",
    }
    return icons.get(path.suffix, "📄 ")

def add_folder_to_tree(tree, path, sort_by="name"):
    """Add a folder to the tree, handling links and sorting."""
    style = "dim" if path.name.startswith("__") else ""
    # Codificar la URL de la carpeta usando quote() para manejar espacios
    branch = tree.add(
        f"[bold magenta]:open_file_folder: [link file://{quote(str(path))}]"
        f"{escape(path.name)}",
        style=style,
        guide_style=style,
    )
    return branch

def add_file_to_tree(tree, path):
    """Add a file to the tree, with icon and size."""
    text_filename = Text(path.name, "green")
    text_filename.highlight_regex(r"\..*$", "bold red")  # Corrección aquí
    # Codificar la URL del archivo usando quote() para manejar espacios
    text_filename.stylize(f"link file://{quote(str(path))}")
    file_size = path.stat().st_size
    file_modified = path.stat().st_mtime
    file_modified = datetime.fromtimestamp(file_modified)   
    file_modified = file_modified.strftime("_%m%d_%H%M_")
    file_created = path.stat().st_ctime
    text_filename.append(f" ({decimal(file_size)})", "blue")
    text_filename.append(f" ({str(file_modified)})", "cyan")  
    icon = get_icon(path)
    tree.add(Text(icon) + text_filename)

def walk_archive(archive, tree: Tree, sort_by="name"):
    """Recursively build a Tree with archive contents."""
    nodes = {}
    for member in archive.infolist():
        path = pathlib.Path(member.filename)
        if not path.name:
            continue
        parent = str(path.parent)
        if parent == ".":
            parent_node = tree
        else:
            parent_node = nodes.get(parent)
            if not parent_node:
                parent_parts = list(path.parents)[:-1]
                for parent_part in parent_parts[::-1]:
                    parent_str = str(parent_part)
                    if parent_str not in nodes:
                        grandparent = str(parent_part.parent)
                        grandparent_node = nodes.get(grandparent, tree)
                        nodes[parent_str] = grandparent_node.add(
                            f"[bold magenta]:open_file_folder: {escape(parent_part.name)}"
                        )
                parent_node = nodes[parent]
        if member.is_dir():
            nodes[str(path)] = parent_node.add(
                f"[bold magenta]:open_file_folder: {escape(path.name)}"
            )
        else:
            text_filename = Text(path.name, "green")
            # text_filename.highlight_regex(r"\..*$", "bold red")  # Corrección aquí
            text_filename.highlight_regex(r"\.([^.]*)$", "bold red") # mod_2710_1554
            text_filename.append(f" ({decimal(member.file_size)})", "blue")
            
            # Obtener la fecha de modificación del archivo en el archivo comprimido 
            # file_modified = member.date_time
            # formatted_datetime = file_modified.strftime("_%m%d_%H%M_") # mod_2710_1946
            # text_filename.append(f" ({formatted_datetime})", "cyan")  # Agregar la fecha al texto            
            
            icon = get_icon(path)
            parent_node.add(Text(icon) + text_filename)

def process_archive(archive_path, parent_tree, sort_by="name"):
    """Process a single archive file (zip, rar, or 7z)."""
    try:
        if archive_path.endswith(".zip"):   
            archive = zipfile.ZipFile(archive_path, "r")
        elif archive_path.endswith(".rar"):
            archive = rarfile.RarFile(archive_path, "r")
        elif archive_path.endswith((".7z", ".7zip")):
            archive = py7zr.SevenZipFile(archive_path, "r")
        else:
            return  # Ignorar archivos que no sean zip, rar o 7z

        # Crear un nuevo árbol para el archivo comprimido
        archive_tree = Tree(
            f"[bold blue]:open_file_folder: {escape(os.path.basename(archive_path))}"
        )
        walk_archive(archive, archive_tree, sort_by)
        # Añadir el árbol del archivo comprimido al árbol padre
        parent_tree.add(archive_tree)

    except Exception as e:
        print(f"Error al procesar el archivo: {e}")
    finally:
        if archive:
            archive.close()

def walk_directory(directory: pathlib.Path, tree: Tree, sort_by="name") -> None:
    """Recursively build a Tree with directory contents."""
    paths = sorted(
        pathlib.Path(directory).iterdir(),
        key=lambda path: (
            path.is_file(),
            {
                "name": path.name.lower(),
                "modified": path.stat().st_mtime,
                "created": path.stat().st_ctime,
                "size": path.stat().st_size,
                "type": path.suffix.lower(),
            }[sort_by],
        ),
    )
    for path in paths:
        # if path.name.startswith("."): # mod_2710_1045  --- Xa carpetas ocultas---
        #     continue
        if path.is_dir():
            branch = add_folder_to_tree(tree, path, sort_by)
            walk_directory(path, branch, sort_by=sort_by)
        elif path.is_file() and path.suffix in [
            ".zip",
            ".rar",
            ".7z",
            ".7zip",
        ]:
            process_archive(str(path), tree, sort_by)
        else:
            add_file_to_tree(tree, path)

def tr():
    """Generate a folder structure from the specified directory."""
    directory = input("Introduce la ruta de la carpeta: ")
    directory = directory.strip()
    directory = directory.strip('"')

    sort_option = input(
        "Ordenar por (n (name), m (modified), c (created), s (size), t (type)): "
    ).lower()
    sort_by = {
        "n": "name",
        "m": "modified",
        "c": "created",
        "s": "size",
        "t": "type",
    }.get(sort_option, "name")

    tree = Tree(
        f":open_file_folder: [link file://{quote(directory)}]{directory}",
        guide_style="bold bright_blue",
    )
    walk_directory(pathlib.Path(directory), tree, sort_by=sort_by)

    console = Console(record=True)
    console.print(tree)

    timestamp = datetime.now().strftime("_%m%d_%H%M_") # mod_2710_1045
    html_filename = f"estructura_directorio{timestamp}.html"
    ruta_completa = obtener_ruta_completa(directory, html_filename) # mod_2710_1045
    console.save_html(ruta_completa) # mod_2710_1045    
    # console.save_html(html_filename)

    with open(ruta_completa, "r", encoding="utf-8") as f:
        html_content = f.read()
    html_content = html_content.replace(
        "<body>", '<body style="background-color: #282907;">'
    )
    with open(ruta_completa, "w", encoding="utf-8") as f:
        f.write(html_content)

    print(f"Salida guardada en {html_filename}")






if __name__ == "__main__":
    tr()
```


---
<br>
<br>
<br>

## El Output :

![image](https://github.com/user-attachments/assets/b86f801e-c477-4bcb-aac2-204772a3aeb3)


---
<br>
<br>
<br>
<br>




