## Choosing Your Flavor of SDR++

There are several forks of SDR++ available, each with its own features and updates. Here are a few choices for you to consider:

- **Official SDR++**: [https://www.sdrpp.org/](https://www.sdrpp_org/)
- **Official Github**: [https://github.com/AlexandreRouma/SDRPlusPlus](https://github.com/AlexandreRouma/SDRPlusPlus)
- **SannySanoff's Brown Fork**: [https://github.com/sannysanoff/SDRPlusPlusBrown](https://github.com/sannysanoff/SDRPlusPlusBrown)

### Configuring SDR++ and Solving Audio Issues

After installing and starting SDR++ (sdrpp), you may notice your sound device is being used by the application. Stopping SDR++ should restore sound functionality. However, services like YouTube might behave strangely, and a logout/login might be required to fully restore audio. Finding a specific command to reset the audio can be a more convenient workaround.

### Quick Setup Video

For a visual guide on how to set up SDR++ (sdrpp), you can refer to the following video (this one explains the server part as well):
[https://www.youtube.com/watch?v=CkSiaECeZEM&t=157s](https://www.youtube.com/watch?v=CkSiaECeZEM&t=157s)

### Connecting to the Server

The most important step obviously is to make sure you're connecting to the server you just got running. Follow these steps in the `Module Manager` section of sdrpp:

1. Ensure your server is listed (It should be already). If it's not, add it by clicking to drop-down arrow and choosing rtl_sdr_source & give it a name on the left side and then click the plus (+) button.

   ![Image](https://github.com/user-attachments/assets/3ccb7ee5-2d30-438c-8978-08d1b217e59f)   ![Image](https://github.com/user-attachments/assets/e939cabe-b9d7-4fa8-b170-34dc2b7b551c)

2. Go back to the top and edit source and choose the source for your server.  Add your server's IP address and ensure the port matches (you can adjust the port at the bottom if necessary).

    ![Image](https://github.com/user-attachments/assets/ec59c36b-d62f-4b22-a46d-0fdd3ed71ba3)

3. Configure your settings. You can start with the ones shown in the image directly above for reference.  Audio shows the signal levels and I usually have it on that option.  If nothing else, its at least a place to start.      

### Testing and Usage

Give your setup a spin and test it out! Using the server method, you should be able to connect from anywhere, provided you use a VPN. You can also utilize different SDR devices and antennas with it. Happy hunting!
