## Install Miniconda.

Connect to the frontend

```bash 
ssh adrianodif@recas.ba.infn.it
```

and in the terminal just use

```bash
curl -sL "https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh" > "Miniconda3.sh"
```
then

```bash
bash Miniconda.sh
```
and just install it in your preferred path. When asked 
```
Do you wish the installer to initialize Miniconda3
by running conda init? [yes|no]
```

answer `yes`. This will create an automatic configuration in the ~/.bashrc file that will launch miniconda whenever you will login again. Sincerely, I don’t like it so what I do is wrapping this piece of script in ~/.bashrc in a function I can call whenever I want

```bash
function miniconda_setup()
{
__conda_setup="$('/lustre/home/adrianodif/Tools/miniconda3/bin/conda' 'shell.bash' 'hook' 2> /dev/null)" #this path is the one you chose, will be written automatically by miniconda installer
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/lustre/home/adrianodif/Tools/miniconda3/etc/profile.d/conda.sh" ]; then 
        . "/lustre/home/adrianodif/Tools/miniconda3/etc/profile.d/conda.sh"
    else
        export PATH="/lustre/home/adrianodif/Tools/miniconda3/bin:$PATH" 
    fi
fi
unset __conda_setup
}
```

the `__conda_setup` part will be already there, just wrap it in a function.

## Environment Setup

Now open a new connection to the frontend and use

```bash
miniconda_setup 
```

from now on you will see this in your shell

```bash
(base) [adrianodif@ui03 ~]$ 
```

that means that your conda *base* environment is active. Just to be safe run

```bash
conda update -n base -c defaults conda
```

to update conda to the latest conda version.

Let’s now create a new environment

```bash
conda create -n new python=3
```

And you can activate it with

```bash
conda activate new
```

You can check all the environments you have created with

```bash
conda env list
```

To get out of an environment use 

```bash
conda deactivate
```

If you do it in your base environment this will exit conda.

In your new environment let install jupyter-notebooks

```bash
conda install jupyter
```

## Notebook on the HPC machines

On the frontend (`ui02` or `ui03`) open an interactive connection with an HPC machine we need to use `condor_submit`. For a comprehensive guide to HTCondor see [here](https://htcondor.readthedocs.io/en/latest/users-manual/submitting-a-job.html#). Write in a plain file with:

```bash
output=out #output of your job will be written here
error=err #errors 
log=log #log of condor_submit

request_cpus=1
request_memory=4096 #in MB

rank=Memory
queue
```

you can easily do it with `echo`:

```bash
echo "output=out
error=err
log=log

request_cpus=1
request_memory=4096 

rank=Memory
queue" > job
```

Then use `condor_submit` to send the job with `-interactive` to do have the shell accessible interactively.

```bash
condor_submit job -interactive
```

This will let you in a machine with a GPU, 1 CPU reserved to you and 4096MB of memory. E.g., I’ve had acess to `wn-gpu-8-3-22`

```bash
[adrianodif@wn-gpu-8-3-22 ~]$ 
```

And I start my notebeook here.

```bash
miniconda_setup
conda activate new
```

If it is your first time with a notebook in Recas use

```bash
jupyter notebook --generate-config
jupyter notebook password
```
to generate a password. 

Then run:
```bash
jupyter notebook --no-browser --port=8000
```

note that if the port is not available jupyter will warn you that it has moved the notebook to another port and you will have to use it in the next step in place of 8000.

## SSH Tunneling to HPC machine

So on a new terminal you need to access the frontend and open a tunnel from the frontend to that machine:
```bash
ssh -N -f -L localhost:2812:localhost:8000 hpc-gpu-1-4-1
```
In this case since the frontend is quite crowded try to use an unique numeber (e.g. your birthday as I did here).  If the port is not free ssh will warn you and you will have to change it. This step is needed since the machines on the HPC are not reacheable directly but only through the ui02/ui03 bastions.

Last step. On you local machine open a terminal and open a tunnel to the frontend
```bash
ssh -L 2812:localhost:2812 -N -f adrianodif@ui03.recas.ba.infn.it
```
Note that you have to adjust ui02 or ui03 depending on the frontend machine you ended up.

Now if you open, with any browser, the

http://localhost:2812

And you will see something like:

<img width="1677" alt="Schermata 2022-06-20 alle 11 59 35" src="https://user-images.githubusercontent.com/16901146/174580040-c75bf269-7241-4ace-8465-b2cd5b61d5e3.png">

Insert the password you have set above. And you are all set:

<img width="1677" alt="Schermata 2022-06-20 alle 12 00 15" src="https://user-images.githubusercontent.com/16901146/174580057-093bc02b-a690-40f6-81f8-1761fa5c4fae.png">

Note that the folder on which you will navigate is the one in which you started the notebook.

