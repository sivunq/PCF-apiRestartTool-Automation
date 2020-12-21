#Import required modules
import subprocess,time,os,openpyxl,datetime

emailId=""
loginUrl=""

def getUsername():
    dumpUser="Dump/username.txt"
    subprocessCmd("echo %USERNAME% > "+dumpUser)
    fileR = open(dumpUser,'r')
    lines=fileR.read().splitlines()
    psid=lines[0]
    fileR.close()
    os.remove(dumpUser)
    print("******************************Welcome User "+str(psid)+"******************************"+"\n")
    return psid

#check validity of PAR
def checkPar(par):
    print("Checking PAR credentials...")
    parCr=False
    if(par=='""'):
        return parCr
    elif (par=='" "'):
        return parCr
    
    dumpPar="Dump/checkPar.txt"
    checkPar="cf login -a "+loginUrl+" -u "+emailId+" -p "+par+" >> "+dumpPar
    subprocessCmd(checkPar)
    
    fileR = open(dumpPar,'r')
    lines=fileR.read().splitlines()
    if(lines[2]=="OK"):
        parCr=True
    fileR.close()
    os.remove(dumpPar)
    
    return parCr

#send commands to restart APIs
def restartApi(instance,login,apiNames,apiInstances):
    commandString=login
    for j in range(0,len(apiNames)):
        if(instance<apiInstances[j]):
            api=apiNames[j]
            commandString+=" & "+cmdRestart+" "+api+" "+str(instance)
            
    subprocessCmd(commandString)

#send commands to CLI tool 
def subprocessCmd(command):
    process = subprocess.Popen(command,stdout=subprocess.PIPE, shell=True)
    process.communicate()[0].strip() 
	
def getApiInstances(commandString,centre):
	#send commands to get instances of apis
	subprocessCmd(commandString)
	
	#check if APIs exist in data centre, if yes then get it's instance otherwise put in faultyApis list
	faultyAPIs=[]
	apiInstances=[]
	apiExists=False
	fileR = open(fileDump,'r')
	lines=fileR.read().splitlines()
	for n in range(0,len(lines)-1):
		#if API found in data centre
		if(lines[n].startswith("Showing") & lines[n+1].startswith("OK")):
			apiExists=True
			api=lines[n].split(" ")[6]
			print("API "+api+" found in data centre "+centre)
		#if API not found in data centre
		elif(lines[n].startswith("Showing") & (not lines[n+1].startswith("OK"))):
			apiExists=False
			api=lines[n].split(" ")[6]
			print("Some Error with API "+api)
			api=api+"\n"
			faultyAPIs.append(api)
			apiInstances.append(0)
		#if command failed because of some reason
		elif(lines[n].startswith("FAILED")):
			apiExists=False
			api=lines[n+1].split(" ")[1]
			print("API "+api+" not found in data centre "+centre)
			api=api+"\n"
			faultyAPIs.append(api)
			apiInstances.append(0)
			
		if(apiExists==True):
			if(lines[n].startswith("instances")):
				apiInstances.append(int(lines[n].split(":")[1][1]))      
	fileR.close()
	os.remove(fileDump)
	return faultyAPIs,apiInstances
	
def restartInstanceWise(faultyAPIs,apiInstances,centre,apiNames,login,fileLog):
	#proceed only if apis are in running state   
	if(len(apiInstances)>0 and max(apiInstances)>0):
		#proceed instance by instance
		for instance in range(0,max(apiInstances)):
			apisRestarted=[]
			#restart api for particular instance
			print("Restarting Instance-"+str(instance)+" in data centre "+centre)
			restartApi(instance,login,apiNames,apiInstances)
			print("Sent Commands to Restart Instance-"+str(instance)+" in data centre "+centre+"\n")
			
			#check if instance just restarted is up & running or not
			running=previousInstanceStatus(instance,login,apiNames,apiInstances,apisRestarted)
			
			#if instance is not restarted yet, recheck same for 3 times, otherwise move to next API's
			if(running==False):
				tries=1
				while(tries<=3):
					print("Trying Again-"+str(tries))
					running=previousInstanceStatus(instance,login,apiNames,apiInstances,apisRestarted)
					if(running==True):
						break
					tries+=1
			if (running==True):
				print("Instance-"+str(instance)+" of all APIs in data centre "+centre+" restarted successfully"+"\n")
				fileLog.write("Restarted Instance-"+str(instance)+" of following APIs:"+"\n")
				fileLog.writelines(apisRestarted)
				fileLog.write("\n")
			#if api instance is not started yet again after trying three times
			elif(running==False):
				print("Unable to restart Instance-"+str(instance)+" in data centre "+centre+"\n")
				print("Stopping Api Restart Process in data centre "+centre)
				break
		#capture apis which are not found in PCF
		if(len(faultyAPIs)>0):
			fileLog.write("Couldn't Restart following APIs:"+"\n")
			fileLog.writelines(faultyAPIs)
			fileLog.write("\n")
			
	#if API is stopped i.e no instances for any API
	elif(len(apiInstances)>0 and max(apiInstances)==0):
		print("It seems some of APIs provided in data centre "+centre+" are stopped or not found")
		if(len(faultyAPIs)>0):
			fileLog.write("Couldn't Restart following APIs:"+"\n")
			fileLog.writelines(faultyAPIs)
			fileLog.write("\n")
	
#check if previous instance is restarted or not 
def previousInstanceStatus(instance,login,apiNames,apiInstances,apisRestarted):
    print("Waiting for 100Secs for Instance-"+str(instance)+" of all APIs to go Up on PCF")
    secs=100
    while(secs>0):
        print(str(secs)+" secs to go...")
        time.sleep(5)
        secs=secs-5
    print("Checking status Of Instance-"+str(instance)+"\n")
    running=True
    commandString=login
    for j in range(0,len(apiNames)):
        if(instance<apiInstances[j]):
            api=apiNames[j]
            commandString+=" & "+cmdStatus+" "+api+" >>"+fileDump

    subprocessCmd(commandString)
    
    fileR = open(fileDump,'r')
    lines=fileR.read().splitlines()
    for n in range(0,len(lines)):
        if(n<len(lines)-1):
            if(lines[n].startswith("Showing") & lines[n+1].startswith("OK")):
                api=lines[n].split(" ")[6]
            elif(lines[n].startswith("Showing") & (not lines[n+1].startswith("OK"))):
                api=lines[n].split(" ")[6]
                print("Some Error with API-"+api)
                continue
        if(lines[n].startswith("#")):
            instanceApi=lines[n].split(" ")[0][1]
            if(instanceApi==str(instance)):
                status=lines[n].split(" ")[3]
                if(status=="running"):
                    print("Instance-"+str(instance)+" of API-"+api+" has been restarted...")
                    api=api+"\n"
                    apisRestarted.append(api)
                elif(status!="running"):
                    running=False
                    print("**Instance-"+str(instance)+" of API-"+api+" has not restarted yet...")
    fileR.close()
    os.remove(fileDump)
    return running

#Data Centres
centres={
	"WGDC100" : " ",
	"WGDC101" : " ",
	"SYDC100" : " ",
	"SYDC101" : " ",
	"SKM100" : " ",
	"SKM101" : " ",
	"TKO100" : " ",
	"TKO101" : " ",
	"VH100" : " ",
	"NL100": " ",
	"VH101": " ",
	"NL101": " "
}

cmdStatus="cf app"
cmdRestart="cf restart-app-instance"
fileDump="Dump/RestartData.txt"
excelFile="Api Names.xlsx"

#Main Function
def main():
    
    #get username
    psid=getUsername()
    
    #accept par password
    par=input("Please enter PAR:\n")
    #par = getpass("Please enter PAR:\n")
    par='"'+par+'"'
    parCr=checkPar(par)
    
    #check validity of PAR password, proceed only if PAR is valid.
    if(parCr==False):
        print("PAR Credentials Not Correct..."+"\n")
        input("Press enter to exit...")
        #quit()
        return
    elif(parCr==True):
        print("PAR Credentials Correct..."+"\n")
        currentTime=str(datetime.datetime.now())
        currentTime=str(currentTime[8:10])+"-"+str(currentTime[5:7])+"-"+str(currentTime[0:4])+" _"+str(currentTime[11:13])+"h-"+str(currentTime[14:16])+"m-"+str(currentTime[17:19])+"s"
    	
    	#Initialize log file
        logFileName="Logs/ApiRestartLogs__"+currentTime+".txt"
        fileLog = open(logFileName,'w')
        fileLog.write("User-"+str(psid)+"logged into API Restart Tool at "+currentTime+"\n"+"\n")
        
    	#open excel sheet having API names
        try:
            wbRestart = openpyxl.load_workbook(excelFile)
            colApis= openpyxl.utils.cell.column_index_from_string('A')
        except FileNotFoundError:
            print("Excel Sheet RestartApis.xlsx not Found")
            input("Press enter to exit...")
            return
            
        #proceed for each data centre
        for centre in centres:
            apiNames=[]
            sheetRestart=wbRestart[centre]
    
            login="cf login -a "+centres[centre]+" -u "+emailId+" -p "+par
            commandString=login    
        
            #read apis from data centre sheet
            apiRow=2
            while((sheetRestart.cell(row=apiRow, column=colApis).value != None)):
                api=sheetRestart.cell(row=apiRow, column=colApis).value
                if (not api.isspace()):
                    apiNames.append(api)
                    commandString+=" & "+cmdStatus+" "+api+" >> "+fileDump
                apiRow=apiRow+1
            
            #check if there are APIs to be restarted in that data centre sheet
            if(len(apiNames)>0):
                print("Getting number of instances of the APIs from data centre "+centre)
                fileLog.write("*********************************Data Centre "+centre+"*********************************"+"\n"+"\n")
    			
    			#check and get api instances
                faultyAPIs,apiInstances=getApiInstances(commandString,centre)
    				
    			#restart api instance-wise
                restartInstanceWise(faultyAPIs,apiInstances,centre,apiNames,login,fileLog)
                
                fileLog.write("*************************************************************************************"+"\n"+"\n")
				
            #if no API to restart in the excel sheet
            else:
                print("No APIs to Restart in data centre "+centre) 
                fileLog.write("No APIs to Restart in data centre "+centre+"\n"+"\n")          
        
        fileLog.write("***********************************END****************************************"+"\n"+"\n")
        fileLog.close()
        wbRestart.close()
        print("Please Check Log file "+logFileName)    
        input("Press enter to exit...")

if __name__ == "__main__" : main()
