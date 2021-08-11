# Install a GUI locally on Proxmox

**This is to show how to install the GUI locally on a proxmox server to have local web UI access. It works like a web kiosk**

**If you want to add a non-root user to auto-login as instead of putting root**

    adduser kiosk-user

Complete the add user steps, you can put in a password but it wont be needed for auto-login

**Install these packages:**

    apt install --no-install-recommends xorg openbox lightdm chromium pulseaudio


**configure lightdm for autologin**

edit /etc/lightdm/lightdm.conf goto section shown below, uncomment autologin-user

    [Seat:*]
    autologin-user=<root here or kiosk-user from above>

**configure openbox to start chromium automatically**

edit "/etc/xdg/openbox/autostart" and add these lines:

    xset -dpms
    xset s off
    chromium --no-sandbox --kiosk https://localhost:8006