# osrm-server
There are many tutorials on how to set up OSRM on your own server, but nothing out there for an absolute beginner. This tutorial will guide you through all the steps to getting OSRM to work on your own server.

Note: You should already have Docker set up on your machine. We'll walk through the steps within it though. 

1. Get the OSM file of your desired area. You can do this one of three ways: 
- Export the OSM file using OSM's [bounding box export feature](https://www.openstreetmap.org/export#map=5/51.500/-0.100)
- Get the link to the OSM file using [Geofabrik](https://www.geofabrik.de/data/download.html)
- If you can't find it above or if your area is too big, you'll need to use [BBBike](https://download.bbbike.org/osm/). The export will download as a ".osm.pbf" file. 

2. If you exported from BBBike, save this file locally and re-upload it to some public link where you can reach it. If you have a direct link to the OSM file, just keep that link. You'll use the link in a few steps to build a Docker instance. 

3. Open Atom or other text editor. Copy and paste the following code: 

```
FROM osrm/osrm-backend
WORKDIR /data
RUN apt-get update
RUN apt-get install -y wget
RUN wget https://path/to/your_OSM_file.osm.pbf
RUN osrm-extract -p /opt/car.lua your_OSM_file.osm.pbf
RUN osrm-partition your_OSM_file.osrm
RUN osrm-customize your_OSM_file.osrm
EXPOSE 5000

CMD ["osrm-routed", "--algorithm" , "mld" , "/data/your_OSM_file.osrm"]
```
A few things you need to customize within this block: 
- Change "your_OSM_file" to your file name. Don't mess with the extensions osm, osrm, etc.
- Change the link after *wget* to your link to the OSM file. 

4. Make some new folder locally and save your file in this folder, with the name *Dockerfile*. Don't add **any** extensions, even .txt. Make sure when you save you're saving as "All Files."

5. Open Terminal or cmd. 

6. Navigate to the directory folder where you saved Dockerfile. 

7. Time to start doing real things in Docker. Login using docker by calling `docker login`. Put in your username and password. 

8. Build the image on Docker using the command `docker build -t yourhub/your-backend-image` where `yourhub` is your Docker hub, and `your-backend-image` is the image name you want to use. 
- What did you just do? You ran a bunch of code within the Dockerfile you created which pulled instructions from *osrm-backend*, created a working directory named *data*, pulled your OSM file using *wget*, and ran the three essential commands to unpack the OSM file so OSRM can use it (*osrm-extract*, *osrm-partition*, and *osrm-customize*). You then exposed it on port 5000 (the default backend port), and finally issued the command to start the actual backend server. The great thing about all this is that everything gets done at once, all you need to do later is run a very basic `docker run` command. 

9. You now can run `docker image ls` to see your new image. 

With the backend covered, now it's time to customize your frontend. You need to do this because by default, OSRM will tell the frontend "look for the backend at localhost:5000" which is great if you're just running from your machine, but not useful at all if you want this to be run on AWS. You will need to change localhost to your public static IP, which you can find on your AWS instance. We'll do this now. 

10. Run the original OSRM frontend as a container: `docker run -it -p 9966:9966 osrm/osrm-frontend`

11. View the container name: `docker container ls`
- The container name is something completely inane like `innocuous_desktop` which Docker thinks up by default. Very creative mind there. 
- Make sure you're looking at the correct name! By this point since you've called `docker run` twice (once on the backend and once on the frontend), you'll have two containers. Be sure you're getting the name of the frontend container. 

12. We're going to edit the frontend code now. Call `docker exec -it container_name sh`

13: This takes you to the `sh` shell. You'll be transported to the area `src`. Type `ls` to see what's in that area. 

14. You'll see another folder called `src`. Type `cd src` to navigate there. 

15. We're looking for the file `leaflet_options.js` where we will make the critical change to the host. You'll see it there. 

16. Type `vi leaflet_options.js` to open that JS file. This looks scary, but is just a simple text editor you can use without leaving the Terminal/command line. Very useful when you're editing Docker images directly. 

17. You'll see an area within that file where it says `localhost:5000`. First, navigate to that place using the arrow keys. Yes, very 1980s. Then press `i` to change the cursor so you can actually type. Change `localhost:5000` to your IP, i.e. `1.2.3.4:5000`. Don't change anything else. You can delete text by pressing your `delete` key. 

18. Once you have made that change, very carefully press `esc` and then type `:wq` and Enter to save your changes. This will return you to the shell script prompt. 

19. Type `exit` to exit the prompt and return from the underworld. 

20. You've now saved the frontend container, you need to make these changes permanent as a new *image*. Type `docker commit container_name yourhub/your_frontend_image`. 

21. Push both new images to your Docker hub! `docker push yourhub/your_frontend_image` and `docker push yourhub/your_backend_image` 

22. Finally, sign in to your AWS instance and go to the terminal. You'll need to use `sudo docker` for all the commands which you previously just had to say `docker` for. 

23. `sudo docker login` to login to your Docker hub from AWS. (You might not actually need to do this.)

24. Pull those images! `sudo docker pull yourhub/your_frontend_image` and `sudo docker pull yourhub/your_backend_image`

25. You should now have both your images on AWS, check using `sudo docker image ls`

Order may not actually matter for these next steps, but try both. 

26. Make a backend container using `sudo docker run -t -i -p 5000:5000 yourhub/your_backend_image`
- It will say "waiting for requests..." if everything worked correctly. But you're not done yet, it's not very user-friendly at this point. 

27. Use CTRL+C to exit, all you did was make that container. You don't need it to be in your way when you make the frontend. This step will take you back to the prompt. 

28. Make a frontend container using `sudo docker run -it -p 9966:9966 yourhub/your_frontend_image`. This may take a few seconds if it's the first time. 

29. Now, check both your containers are running. Use `docker container ls` to see your containers. 
- Frontend should be running on port `9966`. 
- Backend should be running on port `5000`. 

30. In AWS, make sure your firewall isn't blocking those ports. Go to Networking and add two custom ports with the TCP designation. One port will be 9966, the other will be 5000. Save those changes. 

31. Back in your AWS terminal, stop your backend container by calling `sudo docker stop backend_container_name`. `backend_container_name` will have a similarly ridiculous name like `blueberry_feinstein`. Again, make sure you're stopping the backend container, not the frontend one. 

32. Re-run the backend container so that you get the "waiting for requests..." again. 

33. Finally, moment of truth. Open your browser and navigate to `1.2.3.4:9966` where `1.2.3.4` is your IP. Your browser will always point to 9966 (the frontend), don't change that. 

34. Click around and you'll be able to make a route on your own server!



