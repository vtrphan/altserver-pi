# Altserver-Pi
[AltServer](https://github.com/altstoreio/AltStore) docker container for raspberry pi 4

## Steps
1. **Install `docker` and `docker-compose` (running on Pi)**
    - Install [`docker`](https://get.docker.com) and [`docker-compose`](https://docs.docker.com/compose/install/linux/#install-the-plugin-manually)
      
2. **Clone this repo to home folder on Pi (running on Pi)**
    ```bash
    git clone https://github.com/leapbtw/altserver-pi.git && cd altserver-pi
    ```

3. **Install pairing software if necessary (on Mac)**
    - Install `libimobiledevice` and `usbmuxd`.
    - This software is only needed once for pairing, you can perform these steps on a different machine and copy the files to the raspberry later. Package names may vary between distributions, so please check the appropriate package names for your system.

4. **Pair Your iDevice (running on Mac)**
    - Connect your iDevice to your Mac and run:
      ```bash
      idevicepair list # run and check your iDevice, might ask for passcode
      idevicepair pair
      ```
    - Note the UUID when device is successfully paired.
      
5. **Disable `SIP (System Integrity Protection` (Apple Silicon Mac) if necessary in order to access the lockdown folder (running on Mac)**
    - Restart your Mac in Recovery Mode:
         Shut down your Mac.
         Press and hold the power button until you see the startup options window (it should show "Options" with a gear icon).
         Click Options, then click Continue to enter Recovery Mode.
    - Open Terminal in Recovery Mode:
         Once in Recovery Mode, from the Utilities menu, select Terminal.
    - Disable SIP:
         To disable SIP, type the following command in Terminal and press Enter:
      ```bash
      csrutil disable
      ```
      
6. **Copy Pairing Files from Mac to Pi (running on Mac)**
    - Check the files in the lockdown folder (e.g. on M1 Mac):
     ```bash
     sudo ls /var/db/lockdown
     ```
    - Copy the `.plist` files (<uuid>.plist and SystemConfiguration.plist) to the root of the cloned repo on the raspberry (assuming the cloned repo is located in the home folder of the <pi_username>:
      ```bash
      scp /var/db/lockdown/<uuid>.plist <pi_username>@<ip_address_of_pi>:/home/<pi_username>/altserver-pi
      scp /var/db/lockdown/SystemConfiguration.plist <pi_username>@<ip_address_of_pi>:/home/<pi_username>/altserver-pi
      ```
      
7. **Verify Repository Structure (running on Pi)**
    - Your folder structure should look something like this:
      ```bash
      $ ls
      docker-compose.yaml  Dockerfile  provision_config  README.md  startup_script.sh  <uuid>.plist  SystemConfiguration.plist
      ```

8. **Set Permissions (running on Pi)**
    - Set the necessary permissions for the `provision_config` directory:
      ```bash
      chmod 777 provision_config/
      ```

9. **Make sure `libplist` is installed (running on Pi)**
    - libplist is a library used for parsing property lists:
      ```bash
      sudo apt update
      sudo apt install libplist-dev
      ```
    - Verify library installation:
      ```bash
      ls /usr/lib | grep libplist
      ```
    - If nothing is returned, it means the library is not correctly installed or is located somewhere else -> find it:
      ```bash
      sudo find / -name "libplist*"
      ```
      (example of return values from Pi 4):
      ```bash
      /usr/lib/aarch64-linux-gnu/libplist-2.0.so.3.3.0
      /usr/lib/aarch64-linux-gnu/libplist.so
      /usr/lib/aarch64-linux-gnu/libplist-2.0.so
      /usr/lib/aarch64-linux-gnu/libplist.so.3.3.0
      /usr/lib/aarch64-linux-gnu/pkgconfig/libplist.pc
      /usr/lib/aarch64-linux-gnu/pkgconfig/libplist-2.0.pc
      /usr/lib/aarch64-linux-gnu/libplist.so.3
      /usr/lib/aarch64-linux-gnu/libplist-2.0.so.3
      ```
    - Then create symbolic links if necessary:
      ```bash
      sudo ln -s /usr/lib/aarch64-linux-gnu/libplist-2.0.so.3 /usr/lib/libplist-2.0.so.3
      sudo ln -s /usr/lib/aarch64-linux-gnu/libplist-2.0.so /usr/lib/libplist-2.0.so
      sudo ln -s /usr/lib/aarch64-linux-gnu/libplist.so /usr/lib/libplist.so
      ```
    - Then update the system's library cache to include the new links:
      ```bash
      sudo ldconfig
      ```
    - Verify that the libraries are now linked correctly by running:
      ```bash
      ls -l /usr/lib | grep libplist
      ```
      This should show the symbolic links pointing to /usr/lib/aarch64-linux-gnu/

10. **Start the Docker Container**
    - Go to project root folder `altserver-pi`
    - Start the container using Docker Compose:
      ```bash
      docker compose up
      ```

11. **If the `anisette` container still fails to start due to error like `Cannot load any of the following libraries: ["libplist-2.0.so.3", "libplist.so.3", "libplist-2.0.so.4", "libplist.so.4"]`**
    - Try logging into the still running other container e.g. `altserver-pi-altserver-1`:
      ```bash
      docker exec -it <container_name> /bin/bash
      ```
    - Once inside the container's shell, check if the `libplist` libraries are accessible:
      ```bash
      ls /usr/lib/aarch64-linux-gnu | grep libplist
      ```
      You should see the libraries like `libplist.so`, `libplist-2.0.so`, etc. If not, we will need to make sure that the libraries are available in the container:
      - Ensure Volume Mounting of Libraries in Docker:
        You need to mount the host's `/usr/lib/aarch64-linux-gnu/` directory into the container, so the container can access the host's libraries.
        To do this, modify your `docker-compose.yml` file to add a volume mapping for the libraries:
        ```yaml
        services:
          anisette:
            image: "dadoum/anisette-server:latest"
            volumes:
              - ${PWD}/provision_config:/home/Chester/.config/Provision/
              - /usr/lib/aarch64-linux-gnu:/usr/lib/aarch64-linux-gnu:ro  # Mount libraries
            network_mode: "host"
  
          altserver:
            build: .
            network_mode: "host"
            depends_on: ["anisette"]
        ```
        This will mount the libplist libraries from the host system into the container, making them accessible to the application inside the container.
      - After updating the `docker-compose.yml`, you need to Rebuild and Restart the Container:
        ```bash
        docker-compose down
        docker-compose up --build
        ```
      - Verify Libraries Inside the Container:
        ```bash
        docker exec -it <container_name> /bin/bash
        ls /usr/lib/aarch64-linux-gnu | grep libplist
        ```
        If the libraries are now correctly mounted, the application should be able to find and load them.

13. **If all is correct, both containers `altserver-pi-altserver-1` and `altserver-pi-anisette-1` should be up and running

Now you should be able to use the altserver application on your Raspberry Pi 4.

12. **If you want it to run at startup you could create a systemd service like this**
    - Create a new file for your systemd unit:
      ```bash
      sudo vim /etc/systemd/system/altserver.service
      ```
    - Add the following content to the file, *modifying the paths* as needed:
      ```ini
      [Unit]
      Description=AltServer Service
      Requires=docker.service
      After=docker.service

      [Service]
      Type=oneshot
      RemainAfterExit=yes
      WorkingDirectory=/path/to/your/docker-compose.yml
      ExecStart=/usr/local/bin/docker-compose up
      ExecStop=/usr/local/bin/docker-compose down

      [Install]
      WantedBy=multi-user.target
      ```
    - Enable and start the service
      ```bash
      sudo systemctl daemon-reload
      sudo systemctl enable altserver.service
      sudo systemctl start altserver.service
      ```

# Software used in the docker image

- [AltServer-Linux](https://github.com/NyaMisty/AltServer-Linux)
- [anisette-v3-server](https://github.com/Dadoum/anisette-v3-server)
- [netmuxd](https://github.com/jkcoxson/netmuxd)
- [avahi](https://avahi.org/)
