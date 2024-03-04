# A brief review of my new laptop
Recently, I bought a new laptop as my old 8th Gen Dell Inspiron was... tired. It was not only literally breaking in half, but the battery life was measured in minutes and not hours under high load, which it almost always was due to it's weaksauce CPU.

In its stead I got an Acer Aspire 5, specifically the model A515-57-58W1. This is a SKU specific to Brazil, with a few oddities in of itself:

* It has no Windows license out of the box (pretty common in Brazil, since pirating Windows is so rampant customers tend to prefer buying a computer with Linux, saving a few bucks then installing a pirated copy of Windows), but instead of shipping with nothing in the drive or a standard Linux distro like Ubuntu, Acer ended up opting instead to ship with a distro I've never heard of: A so called "Linux Gutta", which I couldn't determine if it was an in house distro from Acer (I doubt it)
* The CPU is an Intel Core **i5-12450H**, which is notable due to it being a H-series part. The equivalent American model has an i5-1235U, which is lower performance from it being a U-series chip.
* It lacks the dedicated Nvidia graphics of the American model, opting to rely on the integrated Intel graphics instead.

So right away this machine has an interesting proposition: Very upgradable with two SODIMM slots, socketed Wi-Fi chip, and dual M.2 NVMe slots, one straight from the CPU at PCI-E Gen 4 x4, and the other from the chipset, running at Gen 3 x4. Due to the fact that it shares the same CPU as its gaming bigger brother, the Nitro 5, and due to it having a Thunderbolt 4 port, it makes it a perfect eGPU gaming machine, which I planned on using it as.

Taking it out the box it was refreshing how almost the whole thing used recyclable materials, mostly cardboard and paper. It slightly concerned me that it didn't immediately turn on, but it took me a minute to realize the machine is configured from the factory to not turn on for the first time without an AC power source connected (much like the Steam Deck's "battery storage" mode, which is fitting since they both share an InsydeH2O BIOS). But right away, I was a little confused.

I booted up Clonezilla to image the "Linux Gutta" off the drive so I can snoop around later (future blog post material 👀) but I noticed that even just sitting on the menus, the fans were spinning pretty fast.

After the imaging begun, the thing sounded like it was gonna take off, yet it was remarkably slow. I figured it must have been some weird Linux oddity (DPTF is infamous for making laptops behave strangely) and I just let it run its course. Once the imaging was complete, in goes the Fedora install media.

After booting into the Fedora Workstation live environment and beginning the installation, I noticed even that was being pretty sluggish and the fans were on overdrive. Confused, I open a terminal window, install lm_sensors, and open the stats, and to my surprise the thing was pinned at 97°C almost constantly, yet drawing quite a bit below the base power target of the CPU.

This tipped me off that something was definitely wrong, and I begun to investigate. Off I go flipping over this two hour old machine, taking the bottom screws off and popping off the bottom cover (and immediately snapping one or two plastic clips), and right away, what I saw simultaneously flabbergasted me and made me drop to the floor laughing:

<img class="mt-4" src="https://files.tabby.page/posts/acer-laptop-review/missing-screws.jpg" alt="No screw to be found"/>

### They forgot the heatsink screws.

Having recovered from the laughter I confirmed that the screws were indeed never installed and didn't just rattle off during shipping by removing the heatsink entirely, revealing the S tier thermal paste spread:

<img class="mt-4" src="https://files.tabby.page/posts/acer-laptop-review/thermal-paste-job.jpg" alt="Amazing spread and coverage"/>

How did this machine pass QC like this? I'm surprised it survived the Clonezilla clone and the Fedora installation processes, I figured the bare part of the die would have burned up and killed it by then? I counted my lucky stars that the CPU came out unscathed, and pressed on.

Obviously the machine wasn't gonna work like this. So off to my bin of random screws I went and sure enough, I found some screws that threaded right in and were just the right size to hold the heatsink down with good pressure. So, I cleaned off the stock goop, and applied a coating of Arctic MX-4.

Screwing it back together and booting the machine up sure enough, thermals were back to normal. Seeing how it was working properly now, I decided against using my warranty and just used the machine as usual, just because, I needed a new laptop *now* (I gave my old one to my little sister), and I couldn't sit around waiting for Acer's RMA processes.

I played around with the now not throttling laptop and immediately it felt much more responsive. It ships with a **Gen4 NVMe SSD (!)** out of the box, and so combined with the high power chip, it resulted in a fast and responsive GNOME desktop on Fedora.

Some more details about this chip: the i5-12450H has a configurable TDP-up of 95W, however Acer for this model has configured it to have a PL2 (peak Turbo Boost power) of 64W, and a PL1 (sustained turbo power) of 45W. Cinebench R23 ran at about 9500 points, with the CPU throttling near the end of PL2 before dropping down to PL1 and running at a toasty but within spec 89°C (more on this later).

Its GPU performance however is quite limited: Usually people associate modern Intel graphics with decent low end grunt, given their new Xe based architecture and Intel's push to compete with AMD on that front. However, this being an H-series, "probably getting installed in a gaming laptop anyway" CPU, it has an "Intel UHD Graphics" GPU as opposed to the "Iris Xe" you see in the U-series chips like the i5-1235U this machine would have in America.

This unfortunately means that the Xe execution units have been cut down significantly, hindering its standalone gaming performance substantially. That didn't stop me though, I did my fair share of gaming on this laptop, mostly lightweight indie titles. The built in HDMI port is HDMI 2.1 though, allowing me to use my 4K TV at its full 60Hz output, while playing Celeste.

The battery life is... it certainly is no ARM MacBook but it's perfectly serviceable for my use. I can go to college and go home four hours later and still have some 20% left in the tank. I don't have a need for a laptop with a massive runtime, just because most places I take it to have a free outlet nearby.

As I used the machine I noticed something was up with the MX-4. It would be fine for a week or two but then the temps would go up and up and up, until even small bursts of high load would nearly immediately throttle the machine. These would be fixed by replacing the paste, only for it to stop cooling properly another two weeks later.

This was highly confusing, because I figured this is probably the paste pumping out, however that should be a months long process, not just over two weeks or so... maybe my less than official mounting scheme was, well, less than ideal. So, I decided to try a curveball:

I logged onto AliExpress and ordered a piece of PTM7950. It is a phase-change, super thin thermal pad, who's performance manages to land somewhere between liquid metal and regular paste for direct-die applications like on a laptop. And boy, did it do wonders. Installing it was a little finicky, it's so thin that it has a tendency to tear as you apply it. However, after I applied it and buttoned the laptop back up, the results were astonishing.

Cinebench results soared up to the 10500 point range, as now thermals were so good, it wouldn't break 90°C throughout the entire PL2 time window, running at a downright chilly (for a laptop) ~80°C sustained all day long, at 45W. What an incredible upgrade, and remarkably inexpensive too for all that extra performance. And hey, it solved the *supposed* paste pumping out issue.

Having used Fedora for a while, I decided to switch to my usual permanent setup for a mobile device: Arch Linux, with root on ZFS, with an encrypted dataset, using a Unified Kernel Image signed with my own custom Secure Boot keys, establishing a chain of trust of sorts, from power on to initramfs (I'll be writing a tutorial on how to install Arch in this highly exotic fashion soon, stay tuned!).

And this is when I found out about a critical flaw with this laptop's BIOS. A bug in Secure Boot and how it fails to honor your custom Secure Boot keys. Stay tuned, next blog post will be about this laptop's broken Secure Boot implementation, and Acer's lack of response.
