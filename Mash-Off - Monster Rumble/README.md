# About
MASH-OFF: Monster Rumble is a local multiplayer twinstick party game about knocking your opponents out of an arena using various cartoony weapons while they try to do the same to you.

Repository: [Link]
Itch: [Link]

# My Responsibilities
<b>Camera</b> 
I worked on a camera that smoothly keeps all players in frame at all times.
The script tracks how far away each player is from each other and calculates how far away the camera needs to be to keep them in frame given it's current FOV and angle.
It moves smoothly and speeds up automatically so players never leave frame. It also keeps max distances caches for a short amount of time to feel less frantic.
Camera shake is also smoothed to help players still be able to track what's going on while adding game juice.
[Code Foldout]

Audio Manager 
I repurposed a AudioManger I wrote for another project which creates and updates an object pool of audio sources for use of soundeffects and music.
If more audio sources are needed the pool is expanded, however so far the sources generated on initalization has always been enough.
The volume of the sources updates whenever the volume settings are changed.
[Code Foldout]

<b>Character Animator</b>
I wrote the character animator which is inspired by this tutorial by youtuber Tarodev
[Embed video https://www.youtube.com/watch?v=ZwLekxsSY3Y]
[Code Foldout]


- I also worked with
<b>Character controlls</b>
<b>The Lobby and Game Start</b>
<b>The Weapon and Pickup system</b>
<b>Main menu UI functionality</b>
