# Enable airpods as headphone
Edit the `ControllerMode` to `bredr` from the default value of `dual`
```bash
sudo emacs /etc/bluetooth/main.conf
```
Then restart the Bluetooth service using
```bash
sudo /etc/init.d/bluetooth restart
```

You should now be able to pair your airpods and use them as headphones.

# Enabling airpods as microphone
1. Install ofono using

    `sudo apt install ofono`
2. Configure pulseaudio to use ofono

    Go to `/etf/pulse/default.pa` 
    
        Find the line `load-module module-bluetooth-discover` 
    
        Change it to `load-module module-bluetooth-discover headset=ofono`
    Add the user `pulse` to the group `bluetooth` to grant the permission 
    
        `sudo usermod -aG bluetooth pulse`
    
    Add the folowing line to `/etc/dbus-1/system.d/ofono.conf` before `</busconfig>`
    
    ```angular2html
    <policy user="pulse">
        <allow send_destination="org.ofono"/>
    </policy>
    ```

3. Provide a "modem" to ofono by installing ofono-phonesim
    
    ```bash
    sudo add-apt-repository ppa:smoser/bluetooth
    sudo apt-get update
    sudo apt-get install ofono-phonesim
   ```
   
    Configure phonesim by adding the following lines to `/etc/ofono/phonesim.conf`

    ```
    [phonesim]
    Driver=phonesim
    Address=127.0.0.1
    Port=12345
   ```
   
    Now, restart the ofono service with 

    ```bash
   sudo systemctl restart ofono.service
    ```
   
4. To run `ofono-phonesim -p 12345 /usr/share/phonesim/default.xml` on startup, create the file `/etc/systemd/system/ofono-phonesim.service` and add the following content.

    ```
    [Unit]
    Description=Run ofono-phonesim in the background
    
    [Service]
    
    ExecStart=ofono-phonesim -p 12345 /usr/share/phonesim/default.xml
    Type=simple
    RemainAfterExit=yes
    
    [Install]
    
    WantedBy=multi-user.target
   ```
   
5. Now, you need to put the modem online. Do this by cloning the following github repos

    ```bash
    cd /tmp
    git clone git://git.kernel.org/pub/scm/network/ofono/ofono.git
    git clone https://github.com/bryanperris/ell.git
   ```
   
    Once you have these repos, `cd /tmp/ofono` and run the following commands

    ```bash
   autoreconf -fi
   ./configure --enable-external-ell
   sudo make
   sudo make install
   ```
   
    If there are any D-Bus errors, install the following
    
    ```bash
   sudo apt install libdbus-1-dev libudev-dev libical-dev libreadline-dev
   ```
   
   Now move the `ofono/` and `ell/` directories to the `/opt/` directory.

   ```bash
   mv ofono/ ell/ /opt/
   ```
    Now you can put the phonesim online by creating another systemd unit file that depends on the ofono-phonesim. Put the following content in `/etc/systemd/system/phonesim-enable-modem.service`

    ```
    [Unit]
    Description=Enable and online phonesim modem
    Requires=ofono-phonesim.service
    
    [Service]
    
    ExecStart=/opt/ofono/test/enable-modem /phonesim
    ExecStart=/opt/ofono/test/online-modem /phonesim
    Type=oneshot
    RemainAfterExit=yes
    
    [Install]
    
    WantedBy=multi-user.target```
   
8. Run both daemons with the following commands

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable ofono-phonesim.service
    sudo systemctl enable phonesim-enable-modem.service
    sudo service phonesim-enable-modem start
   ```
   
    You can check the status of the modem with
    ```bash
    sudo service phonesim-enable-modem status
    ```

    Lastly, restart pulseaudio with `pulseaudio -k`

    
For the modem to be online, you must have `ofono-phonesim -p 12345 /usr/share/phonesim/default.xml` running in a shell.


# References
https://reckoning.dev/blog/airpods-pro-ubuntu/

https://askubuntu.com/questions/1204810/make-no-rule-to-make-target-ell-util-c-needed-by-ell-util-lo

https://askubuntu.com/questions/506116/bluez-installation-configuration-error-for-dbus-1-6

https://github.com/bluez/bluez/issues/125

https://askubuntu.com/questions/1283580/ofono-enable-modem-fails-with-dbus-exceptions-dbusexception-org-ofono-error-fai