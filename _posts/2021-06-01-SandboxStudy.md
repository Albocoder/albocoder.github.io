---
layout: post
title: "Inside commercial malware sandboxes"
categories: malware
---

![Koro Sensei sandbox](https://i.imgur.com/gzO8TuY.jpg)


## Introduction

In this blog post I will explore the commercial malware sandboxes. It appears malware is **allowed** to access the internet in many sandboxes ![KEKW](https://cdn.betterttv.net/emote/5e9c6c187e090362f8b0b9e8/1x). So with that access I decided to collect all the environmental features I could think of and send them to my discord channel. Since Python is the language with the largest library support ever (maybe?) I wrote a python script to take many environment features through `psutil`, `platform`, `cpuinfo` etc. The whole [*python code*](https://gist.github.com/Albocoder/43827f62dceb0970d4810e3719b993d9) is here.

Then we use pyinstaller to create a single executable (~10 MB ![PogO](https://cdn.betterttv.net/emote/5e4e7a1f08b4447d56a92967/1x)) using a command like this: `pyinstaller --onefile --noupx sandbox-env-stealer.py`. Then for the sake of completeness we compressed the executable in the latest upx and uploaded to VirusTotal. After a few crashes we ended with *4* samples:

- [47ed17bdea1dab10fdee…](https://www.virustotal.com/gui/file/47ed17bdea1dab10fdee7f61dff8b8f33ad5d092b3e1e5f0f5a3522a27798183/detection)
- [07a783fc3ae6a065dc0b…](https://www.virustotal.com/gui/file/07a783fc3ae6a065dc0bfad5e8f89ec3ae5be3bb1ec8ada6f774710399f65305/detection)
- [e472c0493a9a35b7975c…](https://www.virustotal.com/gui/file/e472c0493a9a35b7975ca2b5acb4663746993c55c6f2d94742d301d88f050e95/detection)
- [88d38301327da310c5c0…](https://www.virustotal.com/gui/file/88d38301327da310c5c00a0b3ae8209730e033f3c57575a447096d94a647e816/detection) (disregard the file name ![monkaS](https://cdn.betterttv.net/emote/56e9f494fff3cc5c35e5287e/1x))

Of course, you can [download](/assets/misc/sandbox_reports.zip) the data I collected and play for yourself.

## Experiment setup

I submitted one of the first samples on *17/03/2020* (because corona lockdown was getting boring). Then life happened. After deciding to revisit this project I submitted the other 4 samples around March 2021. 
I collected the following features (some ommited for brevity):

- Windows version + arch
- CPU name + core count
- local and internet IP address
- CPU counters
- disk partitions and counters

## What did learn?

### OS version

First, I noticed that **78%** of all the sandboxes run Windows 7 build 7601 (the most famous pirated version ![](https://cdn.betterttv.net/emote/59b73909b27c823d5b1f6052/1x)).
That accounts for 40 out of the 51 executions. The table below illustrates it all. As it seems, the only versions are Windows 7 (build 7600 and 7601) and some flavor of Windows 10.
A malware sample will have a high chance of being run on a real machine if the detected OS is anything but the following.

| OS version       | # execs |
|------------------|---------|
| Win 7 build 7601 |    40   |
| Win 10.0.18362   |    3    |
| Win 7 build 7600 |    3    |
| Win 10.0.14393   |    3    |
| Win 10.0.17134   |    2    |

### Timeline analysis

In this section I will analyze some runtime features of the sandboxes. Here there are some interesting features malware authors can use to quickly identify sandboxes.
There are also some lessons I learned from looking at the executions.

First off, I wanted to see how many executions we were getting per malware and how often. This is particularly important since in [our paper](/assets/pdf_files/malw_variability.pdf) we showed that a malware need to run in 3 random environments at least every 3 weeks.

It appears, for the oldest malware sample **10** executions appear on the same day as the submission, **1** execution 1 day later, **3** executions 33 days later and **2** executions 62 days later.
This means that the sample was deemed interesting 2 months from its first "appearance".
However, we can't conclude that this is what happens for all the samples, with different number of VT engine detections or with more interest from individual AV vendors.
For the record, I did get an execution report back from the first sample just yesterday, after clicking the "*reanalyze*" on VT ![](https://cdn.betterttv.net/emote/5ab6f0ece1d6391b63498774/1x). 
Either way, the malware was executed about 2 to 3 times every month, which is close enough to 3 weeks (that we recommend in our paper), but we demonstrated in the paper that on average 1 week stale of data decreases the detection rate.

### Environment analysis

This sections will show some environment features that the malware can read.

#### The running processes

A common routine seen on many malware and benign samples is that or iterating the running processes.
I (as in python libraries) use a similar routine to retrieve the running processes.
The table below shows some of the running processes and the number of machines they were seen to run on.
Something interesting we can see here are the "special" programs. In some machines we see the appearance of `bitcoin-qt.exe`, `infinium.exe` etc, while in some others we see `steam.exe`, `SteamService.exe` etc, in some others `filezilla.exe` or `centralcreditcard.exe`.
This is usually done to see if the malware is a crypto miner, a game hack, a file infector or a point-of-sale malware respectively. 

<div style="height:300px;overflow:auto;">
<table>
<tr><th>Processes</th><th>Number of machines</th></tr>

<tr><td>...</td><td>...</td></tr>

<tr><td>conhost.exe         </td><td>   47  </td></tr>
<tr><td>lsass.exe           </td><td>   46  </td></tr>
<tr><td>spoolsv.exe         </td><td>   46  </td></tr>
<tr><td>wininit.exe         </td><td>   46  </td></tr>
<tr><td>smss.exe            </td><td>   46  </td></tr>
<tr><td>System Idle Process </td><td>   46  </td></tr>
<tr><td>explorer.exe        </td><td>   46  </td></tr>
<tr><td>winlogon.exe        </td><td>   46  </td></tr>
<tr><td>System              </td><td>   46  </td></tr>
<tr><td>services.exe        </td><td>   46  </td></tr>
<tr><td>opera.exe           </td><td>   43  </td></tr>
<tr><td>firefox.exe         </td><td>   43  </td></tr>
<tr><td>dwm.exe             </td><td>   41  </td></tr>
<tr><td>lsm.exe             </td><td>   40  </td></tr>
<tr><td>Skype.exe           </td><td>   29  </td></tr>
<tr><td>OSPPSVC.EXE         </td><td>   27  </td></tr>
<tr><td>taskeng.exe         </td><td>   24  </td></tr>

<tr><td>...</td><td>...</td></tr>

<tr><td>bitcoin-qt.exe                                                               </td><td>   14 </td></tr>
<tr><td>infium.exe                                                                   </td><td>   14 </td></tr>
<tr><td>qip.exe                                                                      </td><td>   14 </td></tr>
<tr><td>communicator.exe                                                             </td><td>   14 </td></tr>
<tr><td>bitcoind.exe                                                                 </td><td>   14 </td></tr>
<tr><td>steam.exe                                                                    </td><td>   14 </td></tr>
<tr><td>sppsvc.exe                                                                   </td><td>   12 </td></tr>
<tr><td>vslvqrlijtvi.exe                                                             </td><td>   12 </td></tr>
<tr><td>splwow64.exe                                                                 </td><td>   12 </td></tr>
<tr><td>artifact.exe                                                                 </td><td>   10 </td></tr>
<tr><td>fontdrvhost.exe                                                              </td><td>   10 </td></tr>
<tr><td>SteamService.exe                                                             </td><td>    9 </td></tr>
<tr><td>SearchProtocolHost.exe                                                       </td><td>    9 </td></tr>
<tr><td>SearchFilterHost.exe                                                         </td><td>    9 </td></tr>
<tr><td>GoogleUpdate.exe                                                             </td><td>    8 </td></tr>
<tr><td>dllhost.exe                                                                  </td><td>    8 </td></tr>
<tr><td>ioynossujx.exe                                                               </td><td>    8 </td></tr>
<tr><td>wqwupyjrsw.exe                                                               </td><td>    8 </td></tr>
<tr><td>notepad.exe                                                                  </td><td>    8 </td></tr>
<tr><td>taskmgr.exe                                                                  </td><td>    7 </td></tr>
<tr><td>ONENOTEM.EXE                                                                 </td><td>    7 </td></tr>
<tr><td>sihost.exe                                                                   </td><td>    6 </td></tr>
<tr><td>vmtoolsd.exe                                                                 </td><td>    6 </td></tr>
<tr><td>SearchUI.exe                                                                 </td><td>    6 </td></tr>
<tr><td>TrustedInstaller.exe                                                         </td><td>    6 </td></tr>
<tr><td>1a7446534577bab0984f5eb275bdf1f43ed92dfc.exe                                 </td><td>    6 </td></tr>
<tr><td>ivpvkimw.exe                                                                 </td><td>    6 </td></tr>
<tr><td>utg2.exe                                                                     </td><td>    6 </td></tr>
<tr><td>Helios12.exe                                                                 </td><td>    5 </td></tr>
<tr><td>OfficeClickToRun.exe                                                         </td><td>    5 </td></tr>
<tr><td>OmniPOS.exe                                                                  </td><td>    5 </td></tr>
<tr><td>ifs.exe                                                                      </td><td>    5 </td></tr>
<tr><td>EdcSvr.exe                                                                   </td><td>    5 </td></tr>
<tr><td>Registry                                                                     </td><td>    5 </td></tr>
<tr><td>OUTLOOK.EXE                                                                  </td><td>    5 </td></tr>
<tr><td>wmpnetwk.exe                                                                 </td><td>    5 </td></tr>
<tr><td>CentralCreditCard.exe                                                        </td><td>    5 </td></tr>
<tr><td>8lfuaq3.exe                                                                  </td><td>    4 </td></tr>
<tr><td>SophosFileScanner.exe                                                        </td><td>    4 </td></tr>
<tr><td>e5d46536.exe                                                                 </td><td>    4 </td></tr>
<tr><td>SgrmBroker.exe                                                               </td><td>    4 </td></tr>
<tr><td>nvtray.exe                                                                   </td><td>    4 </td></tr>
<tr><td>hmpalert.exe                                                                 </td><td>    4 </td></tr>
<tr><td>350befaf.exe                                                                 </td><td>    4 </td></tr>
<tr><td>1a34b48b.exe                                                                 </td><td>    4 </td></tr>
<tr><td>avp.exe                                                                      </td><td>    3 </td></tr>
<tr><td>SEDService.exe                                                               </td><td>    3 </td></tr>
<tr><td>Tcpview.exe                                                                  </td><td>    3 </td></tr>
<tr><td>mp3tray.exe                                                                  </td><td>    3 </td></tr>
<tr><td>InstallRite.exe                                                              </td><td>    3 </td></tr>
<tr><td>SavService.exe                                                               </td><td>    3 </td></tr>
<tr><td>scap.exe                                                                     </td><td>    3 </td></tr>
<tr><td>mscorsvw.exe                                                                 </td><td>    3 </td></tr>
<tr><td>SAVAdminService.exe                                                          </td><td>    3 </td></tr>
<tr><td>sdrservice.exe                                                               </td><td>    3 </td></tr>
<tr><td>StartMenuExperienceHost.exe                                                  </td><td>    3 </td></tr>
<tr><td>Procmon.exe                                                                  </td><td>    3 </td></tr>
<tr><td>backgroundTaskHost.exe                                                       </td><td>    3 </td></tr>
<tr><td>HttpLog.exe                                                                  </td><td>    3 </td></tr>
<tr><td>ShellExperienceHost.exe                                                      </td><td>    3 </td></tr>
<tr><td>taskhostw.exe                                                                </td><td>    3 </td></tr>
<tr><td>popwack.exe                                                                  </td><td>    3 </td></tr>
<tr><td>msdtc.exe                                                                    </td><td>    3 </td></tr>
<tr><td>sedsvc.exe                                                                   </td><td>    3 </td></tr>
<tr><td>avpui.exe                                                                    </td><td>    3 </td></tr>
<tr><td>procexp64.exe                                                                </td><td>    3 </td></tr>
<tr><td>Procmon64.exe                                                                </td><td>    3 </td></tr>
<tr><td>SophosCleanM64.exe                                                           </td><td>    2 </td></tr>
<tr><td>module-cargo.exe                                                             </td><td>    2 </td></tr>
<tr><td>WindowsInternal.ComposableShell.Experiences.TextInput.InputApp.exe           </td><td>    2 </td></tr>
<tr><td>MemCompression                                                               </td><td>    2 </td></tr>
<tr><td>DS5FEMT81XbOM0LW.exe                                                         </td><td>    2 </td></tr>
<tr><td>SecurityHealthService.exe                                                    </td><td>    2 </td></tr>
<tr><td>88d38301327da310c5c00a0b3ae8209730e033f3c57575a447096d94a647e816.exe         </td><td>    2 </td></tr>
<tr><td>swc_service.exe                                                              </td><td>    2 </td></tr>
<tr><td>KMSAuto Net.exe                                                              </td><td>    2 </td></tr>
<tr><td>3sO7_zsS.exe                                                                 </td><td>    2 </td></tr>
<tr><td>SophosFS.exe                                                                 </td><td>    2 </td></tr>
<tr><td>TiWorker.exe                                                                 </td><td>    2 </td></tr>
<tr><td>s35zi2y.exe                                                                  </td><td>    2 </td></tr>
<tr><td>ld8itap.exe                                                                  </td><td>    2 </td></tr>
<tr><td>SophosNtpService.exe                                                         </td><td>    2 </td></tr>
<tr><td>GoogleUpdateSetup.exe                                                        </td><td>    2 </td></tr>
<tr><td>SSPService.exe                                                               </td><td>    2 </td></tr>
<tr><td>swi_service.exe                                                              </td><td>    2 </td></tr>
<tr><td>pythonw.exe                                                                  </td><td>    2 </td></tr>
<tr><td>WbjiETqs.exe                                                                 </td><td>    2 </td></tr>
<tr><td>ctfmon.exe                                                                   </td><td>    2 </td></tr>
<tr><td>pw.exe                                                                       </td><td>    2 </td></tr>
<tr><td>swi_filter.exe                                                               </td><td>    2 </td></tr>
<tr><td>hltpwzd.exe                                                                  </td><td>    2 </td></tr>
<tr><td>Sophos.Encryption.BitLockerService.exe                                       </td><td>    2 </td></tr>
<tr><td>uniform-98682.exe                                                            </td><td>    2 </td></tr>
<tr><td>8eu3umxnf.exe                                                                </td><td>    2 </td></tr>
<tr><td>SophosIPS.exe                                                                </td><td>    2 </td></tr>
<tr><td>union_rechnung_install_39213.exe                                             </td><td>    2 </td></tr>
<tr><td>userinit.exe                                                                 </td><td>    2 </td></tr>
<tr><td>gzqhbp.exe                                                                   </td><td>    2 </td></tr>
<tr><td>05a62b54.exe                                                                 </td><td>    2 </td></tr>
<tr><td>SophosSafestore64.exe                                                        </td><td>    2 </td></tr>
<tr><td>sdcservice.exe                                                               </td><td>    2 </td></tr>
<tr><td>swi_fc.exe                                                                   </td><td>    2 </td></tr>
<tr><td>3myJvMOn.exe                                                                 </td><td>    2 </td></tr>
<tr><td>WmiApSrv.exe                                                                 </td><td>    2 </td></tr>
<tr><td>cuckoo-47ed17bdea1dab10fdee7f61dff8b8f33ad5d092b3e1e5f0f5a3522a27798183.exe  </td><td>    2 </td></tr>
<tr><td>mtwebooS.exe                                                                 </td><td>    2 </td></tr>
<tr><td>05a62b54e6e32c406f33d22634b03fe8.exe                                         </td><td>    2 </td></tr>
<tr><td>SnrUNWUv.exe                                                                 </td><td>    2 </td></tr>
<tr><td>msiexec.exe                                                                  </td><td>    2 </td></tr>
<tr><td>follow-sneaky-on-twitch.exe                                                  </td><td>    2 </td></tr>
<tr><td>SophosHealth.exe                                                             </td><td>    2 </td></tr>
<tr><td>...</td><td>...</td></tr>
<tr><td>absolutetelnet.exe        </td>   <td> 1 </td>  </tr>                      
<tr><td>gmmeby.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>outlook.exe               </td>   <td> 1 </td>  </tr>                      
<tr><td>isspos.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>qgksae.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>totalcmd.exe              </td>   <td> 1 </td>  </tr>                      
<tr><td>ncftp.exe                 </td>   <td> 1 </td>  </tr>                      
<tr><td>whatsapp.exe              </td>   <td> 1 </td>  </tr>                      
<tr><td>igfxCUIService.exe        </td>   <td> 1 </td>  </tr>                      
<tr><td>winscp.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>coreftp.exe               </td>   <td> 1 </td>  </tr>                      
<tr><td>barca.exe                 </td>   <td> 1 </td>  </tr>                      
<tr><td>socbristol.exe            </td>   <td> 1 </td>  </tr>                      
<tr><td>rundll32.exe              </td>   <td> 1 </td>  </tr>                      
<tr><td>accupos.exe               </td>   <td> 1 </td>  </tr>                      
<tr><td>bedrooms-story-avoid.exe  </td>   <td> 1 </td>  </tr>                      
<tr><td>active-charge.exe         </td>   <td> 1 </td>  </tr>                      
<tr><td>fling.exe                 </td>   <td> 1 </td>  </tr>                      
<tr><td>vrmafl.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>gmailnotifierpro.exe      </td>   <td> 1 </td>  </tr>                      
<tr><td>pidgin.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>diaryrecent.exe           </td>   <td> 1 </td>  </tr>                      
<tr><td>creditservice.exe         </td>   <td> 1 </td>  </tr>                      
<tr><td>operamail.exe             </td>   <td> 1 </td>  </tr>                      
<tr><td>centralcreditcard.exe     </td>   <td> 1 </td>  </tr>                      
<tr><td>unsecapp.exe              </td>   <td> 1 </td>  </tr>                      
<tr><td>AutoKMS.exe               </td>   <td> 1 </td>  </tr>                      
<tr><td>medical reservoir.exe     </td>   <td> 1 </td>  </tr>                      
<tr><td>alftp.exe                 </td>   <td> 1 </td>  </tr>                      
<tr><td>netsh.exe                 </td>   <td> 1 </td>  </tr>                      
<tr><td>wspsvc.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>scriptftp.exe             </td>   <td> 1 </td>  </tr>                      
<tr><td>spgagentservice.exe       </td>   <td> 1 </td>  </tr>                      
<tr><td>slwvdq.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>edcsvr.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>Sysmon.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>american.exe              </td>   <td> 1 </td>  </tr>                      
<tr><td>wmi64.exe                 </td>   <td> 1 </td>  </tr>                      
<tr><td>flashfxp.exe              </td>   <td> 1 </td>  </tr>                      
<tr><td>axdnik.exe                </td>   <td> 1 </td>  </tr>                      
<tr><td>webpagepioneer.exe        </td>   <td> 1 </td>  </tr>                      
<tr><td>skype.exe                 </td>   <td> 1 </td>  </tr>                      
<tr><td>Memory Compression        </td>   <td> 1 </td>  </tr>                      
<tr><td>fpos.exe                  </td>   <td> 1 </td>  </tr>                      
<tr><td>ApplicationFrameHost.exe  </td>   <td> 1 </td>  </tr>                      
<tr><td>filezilla.exe             </td>   <td> 1 </td>  </tr>                      
<tr><td>spcwin.exe                </td>   <td> 1 </td>  </tr>                      
</table>
</div>
<br>

Unfortunately, we can also see things like `cuckoo-47ed17bdea1dab10fdee7f61dff8b8f33ad5d092b3e1e5f0f5a3522a27798183.exe` or `1a7446534577bab0984f5eb275bdf1f43ed92dfc.exe` which is simply the checksum hash of the sample. An attacker can simply compute the popular checksums (the ones on the `details` section in VT) and see if its name is any of those and terminate ![](https://cdn.betterttv.net/emote/5d6096974932b21d9c332904/1x).

#### Machine names

One thing I wanted to see is the username the malware in the sandbox will run on. For the most part I was underwhelmed. It appears the machine name remains the same across executions, which may be easy for an attacker to just submit bogus "malware" just to harvest all the machine names. In terms of the context I did see that sometimes the sample was executed as `Administrator` meaning that some sandboxes give the "malware" admin privilleges. This is done to make sure that malware can execute.

<div style="height:300px;overflow:auto;">
<table>
<tr><th>Username</th><th>Number of executions</th></tr>
<tr><td>art-PC          </td><td> 7 </td></tr>
<tr><td>PC-4a095e27cb   </td><td> 5 </td></tr>
<tr><td>w7sb64-01       </td><td> 3 </td></tr>
<tr><td>w7x64           </td><td> 3 </td></tr>
<tr><td>z97Otih0P4v-PC  </td><td> 2 </td></tr>
<tr><td>AMAZING-AVOCADO </td><td> 2 </td></tr>
<tr><td>mgvwazbfy       </td><td> 1 </td></tr>
<tr><td>WIN-IMCGBF4ZV49 </td><td> 1 </td></tr>
<tr><td>HAPUBWS-PC      </td><td> 1 </td></tr>
<tr><td>DESKTOP-VXO5LFI </td><td> 1 </td></tr>
<tr><td>QGl87k-PC       </td><td> 1 </td></tr>
<tr><td>IOFXBF742797820 </td><td> 1 </td></tr>
<tr><td>WIN-QVM8C8V0B1E </td><td> 1 </td></tr>
<tr><td>XDuwTfOno       </td><td> 1 </td></tr>
<tr><td>dillon          </td><td> 1 </td></tr>
<tr><td>WIN-BROIECEJLD2 </td><td> 1 </td></tr>
<tr><td>CZAC38122213349 </td><td> 1 </td></tr>
<tr><td>Xj8Uz1ljKXdt-PC </td><td> 1 </td></tr>
<tr><td>DESKTOP-ILTLN65 </td><td> 1 </td></tr>
<tr><td>OngJeNyHDSzmoUHw</td><td> 1 </td></tr>
<tr><td>Anna-PC         </td><td> 1 </td></tr>
<tr><td>DcXhlNjDfk-PC   </td><td> 1 </td></tr>
<tr><td>WIN-FYI1QSCHQHU </td><td> 1 </td></tr>
<tr><td>WIN-LQOUJKDIROR </td><td> 1 </td></tr>
<tr><td>XGUW12547433669 </td><td> 1 </td></tr>
<tr><td>AL MUKALLA      </td><td> 1 </td></tr>
<tr><td>WIN-KRAZH63AMC2 </td><td> 1 </td></tr>
<tr><td>PC              </td><td> 1 </td></tr>
<tr><td>GCSPJUXFT667743 </td><td> 1 </td></tr>
<tr><td>DESKTOP-D019GDM </td><td> 1 </td></tr>
<tr><td>WIN-UJ21PNWQMR2 </td><td> 1 </td></tr>
<tr><td>Lisa-PC         </td><td> 1 </td></tr>
<tr><td>vX3juZIWR5Wy-PC </td><td> 1 </td></tr>
<tr><td>WIN-TGCR76AWNUB </td><td> 1 </td></tr>
<tr><td>WIN-U1DY5TBDUI7 </td><td> 1 </td></tr>
</table>
</div>
<br>

### Hardware analysis

#### CPU and memory analysis

Next we are interested to know what CPU are the sandboxes running on?
I realized the `platform.processor()` was not returning the actual CPU name, but it was too late when I realized so I used the available information from this command and scrapped some tables online to get the processor name ([here](/assets/misc/cpu_name_database.pkl) is the *pkl* file that aggregates all data).
In the latest version I added this awesome library called `cpuinfo`, so in case you need to collect your own data the cpuinfo will do the work.

| number of cores  | potential cores |
|------------------|-----------------|
|         1        |    { 8 , 4 }    |
|         1        |    { 8 , 4 }    |
|         1        |    { 8 , 4 }    |
|         4        |     { 16 }      |
|         2        |    { 8 , 4 }    |
|         1        |    { 8 , 4 }    |
|         1        |    { 8 , 4 }    |
|         1        |    { 8 , 4 }    |
|         1        |    { 8 , 4 }    |

As hypothesized, the machine has the wrong number of cores for the CPU name (VICTORY ![](https://cdn.betterttv.net/emote/5f1b0186cf6d2144653d2970/1x)). In the **9** machines, I found that the most used CPUs are `Intel(R) Core(TM) i5-7500 CPU @ 3.40GHz`, `Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz` and `Intel(R) Xeon(R) W-2140B CPU @ 3.20GHz`. The machines seem to have 1 core for the most times but the name is that of a cpu with 4 or 8 cores.

I also checked the CPU utilization rates. Hypothetically, the sandboxes would have low CPU utilization, since they are only meant to run the malware. 

![](/assets/blog1_sandbox/cpu_utilization.svg)

While most of the sandboxes have a utilization less than 20% there are still cases where sandboxes have 80%  or even 100%. I believe the CPU utilization cannot be a feature to distingush between sandboxes and real machines.

Lastly for this section we look into memory. How much memory can commercial sandboxes spare?

![](/assets/blog1_sandbox/memory_in_machines.svg)

It appears most of the machines have 1 to 2 GB of RAM to spare. In 2 cases I found machines with 512MB of memory like its 2010 ![](https://cdn.betterttv.net/emote/55cbeb8f8b9c49ef325bf738/1x). Props to the AV vendor(s) that allocated 4, 8 and 16GB for a sandbox to run a malware sample.

#### Disk Partitions

An interesting result here is that in ALL sandboxes there seems to be 1 disk partion of type `cdrom` ![](https://cdn.betterttv.net/emote/58e5abdaf3ef4c75c9c6f0f9/1x).
Hypothetically this may happen because during the VM creation the Windows installation disk was left inserted (more work needs to be done to verify this).

For the main disk partition (where Windows is installed) we notice quite a spectrum of sizes, but the usage in general is quite low (except for the `32GB` disks with high usage because Windows is installed there). In 2021 I wouldn't expect there to be many machines left with 32GB in the main drive, so this may raise some suspicions from the attackers prespective. 

![](/assets/blog1_sandbox/disk_sizes.svg)
![](/assets/blog1_sandbox/disk_usages.svg)
 
#### Network Interfaces

In general 2 to 3 interfaces and **5** network interfaces in 1 sandbox. The most prevalent are of course `Loopback Pseudo-Interface 1` and `Local Area Connection`.
Then I noticed some pattern of `isatap.{<random GUID here>}` and `Teredo Tunneling Pseudo-Interface`. Upon further analysis I found that these interface exist to enable IPv6 communication, so there is nothing special that an attacker can use here ![](https://cdn.betterttv.net/emote/5d7eefb7c0652668c9e4d394/1x).

#### Battery

Not a single sandbox had a battery. Attackers right now pulling a high IQ strat ![](https://cdn.betterttv.net/emote/5acdc7cb31ca5d147369ead8/1x).

### Time analysis

One thing its hard to keep up to date while restoring the snapshot (at least in Windows) is local time.
Its a bit more difficult when you consider the geolocation the machine is supposed to be at.
For this I collected the local time, the global UTC time and the external IP address of the sandbox and metadata for that IP.
Thanks to [`ipfy`](https://www.ipify.org/), [`just-the-time`](https://just-the-time.appspot.com/), and [`ipinfo`](https://ipinfo.io/) for the awesome service.
The process I followed is pretty simple. I use the data from `ipinfo` to get the geolocation and convert the UTC time to the local time for that geolocation, then I calculated the time skew between the time of the geolocation and the sandboxes local time.


| time difference(hours)  | number of machines |
|------------------|--------------------|
|       -19.0      |          1         |
|       -9.0       |          2         |
|       -8.0       |          1         |
|       -3.0       |          1         |
|       -2.0       |         18         |
|       -1.0       |          4         |
|        **0.0**       |       **19**        |
|        3.0       |          1         |
|        5.0       |          1         |
|        9.0       |          2         |


As it appears, around **65%** of the sandboxes have some sort of a time skew (and I believe the other 35% are simply lucky to VPN on the same timezone or not even VPN at all, will get to this later ![](https://cdn.betterttv.net/emote/59f27b3f4ebd8047f54dee29/1x)).


This is a very low effort feature a real malware can use to check if its inside a sandbox given that such info can be checked for free.
Even through a CnC server the attacker can measure whether the malware has made it into the sandboxes.
Throughout our [`paper`](/assets/pdf_files/malw_variability.pdf) we also noticed that there are some real users' machines with an outdated clock (sometimes out of date by about 50 years ![](https://cdn.betterttv.net/emote/5d20a55de1cfde376e532972/1x)) however this made up for *less than 0.001%* of the machines in the real world not 65%, so a malware author may simply choose to forgive these outdated machines just to be safe from analysis.
And from a defender's prespective, PLEASE UPDATE THE CLOCK BEFORE ROUTING THE NET TRAFFIC TO NARNIA.

### IP analysis

This is the meat of the blog post, in my opinion. Thanks again to [`ipinfo`](https://ipinfo.io/) for the 7 day free trial. As the most interesting to me, I first looked at "Who owns these IP addresses? Who are the companies/ISPs?". As shown in the table below the network traffic in all the sandboxes is routed through VPNs that they get from dedicated servers. It appears CrowdStrike has their own IP range that they use to route traffic (registered under their official name ![](https://cdn.betterttv.net/emote/55f47f507f08be9f0a63ce37/1x), hide the pain Harold).

<div style="height:300px;overflow:auto;">
<table>
<tr><th>IP owner company</th><th>number of IPs in the dataset</th></tr>
<tr><td>TELUS Communications Inc.          </td><td>    5      </td></tr>
<tr><td>Wintek Corporation                 </td><td>    5      </td></tr>
<tr><td>PJSC Vimpelcom                     </td><td>    5      </td></tr>
<tr><td>Verizon Business Special Project   </td><td>    3      </td></tr>
<tr><td>LLC Digital Network                </td><td>    3      </td></tr>
<tr><td>Cox Communications Inc.            </td><td>    3      </td></tr>
<tr><td>Zwiebelfreunde e.V.                </td><td>    2      </td></tr>
<tr><td>Bell Canada                        </td><td>    2      </td></tr>
<tr><td>111250 Russia Moscow SOVINTEL/EDN  </td><td>    2      </td></tr>
<tr><td>1337 Services LLC                  </td><td>    1      </td></tr>
<tr><td>Vodafone D2 GmbH                   </td><td>    1      </td></tr>
<tr><td>Telecom Colocation, LLC            </td><td>    1      </td></tr>
<tr><td>IPG                                </td><td>    1      </td></tr>
<tr><td>111250 Russia MOscow EDN/Sovintel  </td><td>    1      </td></tr>
<tr><td>Bungee Servers SP                  </td><td>    1      </td></tr>
<tr><td><b>CrowdStrike Services</b>        </td><td>  <b>1</b> </td></tr>
<tr><td>Dedicated Servers                  </td><td>    1      </td></tr>
<tr><td>LeaseWeb Netherlands B.V.          </td><td>    1      </td></tr>
<tr><td>ARCOR AG                           </td><td>    1      </td></tr>
<tr><td>Deutsche Telekom AG                </td><td>    1      </td></tr>
<tr><td>Datacamp Limited                   </td><td>    1      </td></tr>
<tr><td>Core-Backbone GmbH                 </td><td>    1      </td></tr>
</table>
</div>
<br>

Now we want to know where exatly are these IPs from. With this much data it's hard to say which IPs are owned by the AV vendors, since they are also just buying VPN access, but I wanted to see where they are buying from. It appears Russia, US, Canada and Germany are the most prominent countries, and in Russia, Moscow seems to be the city with the highest number of IPs. 

![](/assets/blog1_sandbox/countries_piechart.svg)
![](/assets/blog1_sandbox/IP_locations.svg)

## Conclusion

There is no silver bullet to detect sandboxes, but there are some features and bugs ![](https://cdn.betterttv.net/emote/57f540e061ff29ea0eec6f27/1x) the attacker can use to detect them.
On the other hand the sandboxes can also cover these weaknesses. It's all about that arms race ![](https://cdn.betterttv.net/emote/5d0d7140ca4f4b50240ff6b4/1x).