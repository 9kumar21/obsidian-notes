IMAGE
dockermvk/python-img:v1

ID		 DISK USAGE   CONTENT SIZE EXTRAS
6f390823cc3a        203MB         49.7MB    U


IMAGE: Name of the image.

ID: Unique ID of the image. 
    Note: Every New Image Build = New Unique ID

DISK USAGE: Total Image Size (Complete image size + Unique layer 	    size)
CONTENT SIZE/Unique Size: Only the layers the belong exclusively 			  to the image.

Example: Notice both images use the same base image  

                python:3.12-slim
                     153 MB
                    /       \
                   /         \
          Image A            Image B
            50 MB             40 MB

Docker stores the 153 MB only once. So,

Image A

Total
203 MB

Unique
50 MB

because only the application layer belongs exclusively to Image A.

Similarly,
Image B

Total
193 MB

Unique
40 MB

This is why output shows:
203 MB (Total Size)
49.7 MB (Layers Size exclusive to the image)

U/Used: Image is referred/used. 
	Example: If you tag an Image, or running a container   			 from an image that image is called 				 referred/used. Hence you can see U in EXTRA.
		 If you remove tag or container that image is called unreferred/not used. Hence you don't see U in EXTRA.
________________________________________________________________



**Untag an Image:** 

docker rmi < repository-name >:< tag >
Example: docker rmi dockermvk/python-img:v1

1. If the specified tag is the **Only** tag associated with that image, running this command will untag and completely delete the image from your host/local.
2. If the image has multiple tags pointing to it, this operation safely removes only the specified tag while leaving the underlying image and layers completely intact.

Find all **dangling** (untagged) images:
docker images -f "dangling=true"

**Clean up and bulk-delete all untagged images:**
docker image prune



