```python
#!/usr/bin/env python3
import os
import subprocess
import argparse
from pathlib import Path

def check_docker_installed():
    try:
        subprocess.run(["docker", "--version"], check=True, capture_output=True)
    except subprocess.CalledProcessError:
        print("Error: Docker is not installed or not in PATH")
        exit(1)

def build_images_from_folders(directory):
    print(f"\nBuilding Docker images from folders in: {directory}")
    root_path = Path(directory)
    folders = [f for f in root_path.iterdir() if f.is_dir()]

    if not folders:
        print("No subdirectories found for building images")
        return

    for folder in folders:
        dockerfile = folder / "Dockerfile"
        if not dockerfile.exists():
            print(f"  ⚠️ No Dockerfile found in {folder.name}, skipping...")
            continue

        image_name = f"{folder.name}:data-mining"
        print(f"  🛠️ Building {image_name} from {folder.name}...")

        try:
            subprocess.run(
                ["docker", "build", "-t", image_name, str(folder)],
                check=True,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE
            )
            print(f"  ✅ Successfully built {image_name}")
        except subprocess.CalledProcessError as e:
            print(f"  ❌ Failed to build {image_name}: {e.stderr.decode().strip()}")

def save_images_to_tar(output_dir="docker_images_backup"):
    print("\nSaving Docker images to compressed files...")
    os.makedirs(output_dir, exist_ok=True)

    try:
        images = subprocess.run(
            ["docker", "images", "--format", "{{.Repository}}:{{.Tag}}"],
            check=True,
            capture_output=True,
            text=True
        ).stdout.splitlines()
    except subprocess.CalledProcessError as e:
        print(f"Error getting Docker images: {e.stderr.strip()}")
        return

    images = [img for img in images if "<none>" not in img]

    if not images:
        print("No Docker images found to save")
        return

    saved_count = 0
    for image in images:
        repo, tag = image.split(":")
        safe_repo = repo.replace("/", "_")
        filename = f"{safe_repo}-{tag}.tar.gz"
        filepath = os.path.join(output_dir, filename)

        print(f"  💾 Saving {image} to {filename}...")
        try:
            with open(filepath, "wb") as f:
                save_process = subprocess.run(
                    ["docker", "save", image],
                    check=True,
                    stdout=subprocess.PIPE
                )
                compress_process = subprocess.run(
                    ["gzip"],
                    input=save_process.stdout,
                    stdout=f,
                    check=True
                )
            print(f"  ✅ Saved {image}")
            saved_count += 1
        except subprocess.CalledProcessError as e:
            print(f"  ❌ Failed to save {image}: {str(e)}")

    print(f"\nSaved {saved_count}/{len(images)} images to {output_dir}/")

def main():
    parser = argparse.ArgumentParser(
        description="Docker Image Manager: Build images from folders and save existing images"
    )
    parser.add_argument(
        "-b", "--build",
        metavar="DIRECTORY",
        help="Build Docker images from folders in the specified directory"
    )
    parser.add_argument(
        "-s", "--save",
        action="store_true",
        help="Save all Docker images to compressed tar.gz files"
    )
    parser.add_argument(
        "-o", "--output",
        default="docker_images_backup",
        help="Output directory for saved images (default: docker_images_backup)"
    )

    args = parser.parse_args()

    check_docker_installed()

    if args.build:
        build_images_from_folders(args.build)

    if args.save:
        save_images_to_tar(args.output)

    if not args.build and not args.save:
        parser.print_help()

if __name__ == "__main__":
    main()
```

### Features:

1. **Combined Functionality**:

    - Build Docker images from folders (`-b/--build`)
    - Save existing images to compressed files (`-s/--save`)
    - Can run both operations in sequence

2. **Improved Features**:

    - Proper error handling and user feedback
    - Progress indicators with emojis
    - Skip folders without Dockerfiles
    - Handles image names with repositories (converts `/` to `_`)
    - Creates output directory if it doesn't exist

3. **Usage**:

    ```bash
    # Build images from folders and save all images
    ./docker_manager.py -b /path/to/folders -s

    # Only build images
    ./docker_manager.py -b /path/to/folders

    # Only save existing images
    ./docker_manager.py -s

    # Save to custom output directory
    ./docker_manager.py -s -o custom_backup_dir
    ```

4. **Requirements**:

    - Python 3.x
    - Docker installed and running
    - `gzip` available in PATH

5. **Output Files**:
    - Creates files like `nginx-latest.tar.gz`, `ubuntu-20.04.tar.gz` etc.
    - Stores them in `docker_images_backup/` by default

To use this script:

1. Save as `docker_manager.py`
2. Make executable: `chmod +x docker_manager.py`
3. Run with desired options

The script provides clear visual feedback with emoji indicators and handles errors gracefully. It's more robust than the bash versions while maintaining all the original functionality.
