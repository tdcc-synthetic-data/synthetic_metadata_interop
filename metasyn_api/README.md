# Metasyn API prototype 

This folder contains the code for a fastAPI wrapper for metasyn. It exposes two endpoints: 
- /fit-model/ : accepts a CSV file, fits a metasyn model, and returns the model as GMF (JSON)
- /synthesize/ : accepts a fitted model (as JSON) and generates synthetic data as a CSV file

It does not include any authentication or security features, and is intended for local use and as a starting point for further development.

## File description 
- `main.py` contains the API app 
- `input_data.csv` contains a file from an open dataset in the SSH Data Station that is used as a testing example
- `synthetic.csv` contains the resulting synthetic version of the input data

## Instructions 
After installing the required Python packages, you can start the API with the following command: 

```bash
uvicorn main:app --reload
```

You can now see the documentation here: 

```
http://127.0.0.1:8000/docs
```

You can test or use the endpoints there by following the instructions, but it will not allow you to save the resulting JSON and CSV. To save the model and synthetic data files to your disk, run the following commands: 


Fit model (upload CSV, save response):
``` bash
curl -s -F "file=@input_data.csv" http://127.0.0.1:8000/fit-model/ -o fit_response.json
```

Extract the model into a file:
```bash
jq '.model_json' fit_response.json > model.json
```

Build payload (necessary for next step):
```bash
jq -n --slurpfile m model.json '{model_json: $m[0], num_rows: 109}' > payload.json
```

Call synthesize: 
```bash
curl -s -H "Content-Type: application/json" -d @payload.json http://127.0.0.1:8000/synthesize/ -o synth_response.json
```

Extract csv:
```bash
jq -r '.synthetic_data_csv' synth_response.json > synthetic.csv

```



## AI use 
- `main.py` was written with the help of Lumo, the Proton AI assistant. 

## To do:
- add package requirements 
- turn instructions into bash script