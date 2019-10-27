# virtual environment related

## install new environment
`conda create -n environment_name --file spec_file.txt`

## activate an environmnet
`source activate environment_name`

## deactivate an environment
`conda deactivate`

## list environment
`conda env list`
or
`conda info --envs`

## list packages in environment
`conda list`

## remove environment
`conda remove --name environment_name --all`

## install ipython kernel inside the virtual environment
```
 install ipykernel
ipython kernel install --name=environment_name
```