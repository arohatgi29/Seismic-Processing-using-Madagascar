from rsf.proj import *

#################### 
# Fetch the dataset
####################
tgz = '2D_Land_data_2ms.tgz'
Fetch(tgz,'freeusp')

files = ['Line_001.'+x for x in Split('TXT SPS RPS XPS sgy')]
Flow(files,tgz,
     'gunzip -c $SOURCE | tar -xvf -',stdin=0,stdout=-1)

######################## 
# Convert to RSF format
########################
Flow('line tline','Line_001.sgy','segyread tfile=${TARGETS[1]}')

Flow('first','line','window n2=1000' )
Result('first',
       '''
       grey title="First 1000 traces"
       ''')

Flow('firstagc','first','agc rect1= 20 rect2=50')
Result('firstagc',
       '''
       grey title="First 1000 traces after Automatic Gain Control"
       ''')
#       agc rect1=250 rect2=100 |


######################## 
# Update the Headers
########################
lines = {'S':251,'R':782}

# Arrange receiver coordinates
for case in 'SR':
    # X-Y geometry
    Flow(case+'.asc','Line_001.%cPS' % case,
         '''awk 'NR > 20 {print $8, " ", $9}' ''')
    Flow(case,case+'.asc',
         '''
         echo in=$SOURCE data_format=ascii_float n1=2 n2=%d |
         dd form=native 
         ''' % lines[case],stdin=0) 

shots = []
for shot in range(lines['S']):
    line = 'line%d' % shot
    Flow(line,'R','window f2=%d n2=282' % (2*shot))
    shots.append(line)
Flow('rece',shots,'cat axis=3 ${SOURCES[1:%d]}' % len(shots))
Flow('sour','S','spray axis=2 n=282 o=0 d=1')

# convert line in same dimension as sour and rece

Flow('line_0','line','intbin xk=cdpt yk=fldr | window f2=2' )
Flow('tline_0','tline','intbin xk=cdpt yk=fldr head=$SOURCE | window f2=2 ')

# Separate Sx, Sy, Rx, and Ry
Flow('sourx','sour','window n1=1 |  scale dscale=0.001')
Flow('soury','sour','window f1=1 |  scale dscale=0.001')
Flow('recex','rece','window n1=1 |  scale dscale=0.001')
Flow('recey','rece','window f1=1 |  scale dscale=0.001')

# Calculate the offset
Flow('offset','sourx soury recex recey',
     '''
     math SX=${SOURCES[0]} SY=${SOURCES[1]}
     RX=${SOURCES[2]} RY=${SOURCES[3]}
     output="sqrt((RX-SX)^2+(RY-SY)^2)"
     ''')

# change to integers to edit the headers
Flow('sx', 'sourx', 'dd type=int')
Flow('sy', 'soury', 'dd type=int')
Flow('rx', 'recex', 'dd type=int')
Flow('ry', 'recey', 'dd type=int')
Flow('o', 'offset', 'dd type=int')

Flow('header_new','line_0 tline_0 sx sy rx ry o',
     'segyheader tfile=${SOURCES[1]} sx=${SOURCES[2]} sy=${SOURCES[3]} gx=${SOURCES[4]} gy=${SOURCES[5]} offset=${SOURCES[6]}')

# ##############################
#  visualize regular geometry
# ##############################
Flow('lines','line_0','put label3=Source d3=0.05  o3=688  unit3=km  label2=Offset d2=0.025 o2=-3.5 unit2=km label1=Time unit1=s')
Result('lines',
       '''
       transp memsize=1000 plane=23 |
       byte gainpanel=each |
       grey3 frame1=500 frame2=100 frame3=120 flat=n movie=2
       title="Raw Data"
       ''')

# ##############################
#  First break mute
# ##############################

# Shot 100
Flow('shot100','lines','window n3=1 f3=100')
Plot('shot100',' agc rect1=50 rect2=20  | grey title="Shot 100"')

# Select muting parameter for background noise
Flow('mute100','shot100','mutter  slope0=0.2')
Plot('mute100','agc rect1=50 rect2=20 | grey title="Shot 100 after mute"') 

Result('mute100','shot100 mute100','SideBySideAniso')

#Apply mute to all shots
Flow('mutes','lines','mutter slope0=0.2')


# ##############################
# Frequency spectra
# ##############################

# spectra for shot 100
Flow('spec100','mute100','spectra2')
Plot('spec100',' grey color=j title="spectra before subsampling"')
Result('spec100','mute100 spec100','SideBySideAniso')

#subsampling to 4ms
Flow('subsample100', 'mute100', 'bandpass flo=3 fhi=125| window j1=2')
Plot('subsample100','agc rect1=50 rect2=20 | grey title="Shot 100 after subsampling"')
Flow('subspectra100','subsample100','spectra2')
Plot('subspectra100','grey color=j title="Spectra after subsampling"')
Result('subspectra100','subsample100 subspectra100','SideBySideAniso')

# subsampling all shots to 4ms
Flow('subsample', 'mutes', 'bandpass flo=3 fhi=125| window j1=2')


##############################
# FK Filter for GR
##############################
Flow('fk','subsample100','fft1 | fft3')
Result('fks','fk',
       '''
       real | 
       grey color=j 
       title="F-K spectra"
       ''')
Flow('rmutter','fk','real | mutter slope0=3 ')
Flow('imutter','fk','imag | mutter slope0=3 ')
Plot('rfks','rmutter',
       '''
       grey color=j
       title="Muted F-K spectra"
       ''')
Flow('mutter','rmutter imutter','cmplx ${SOURCES[:2]}')
Flow('inoi','mutter',
     '''
     fft3 inv=y |
     fft1 inv=y |
     mutter slope0=3 inner=n
     ''')
Plot('inoi',' agc rect1=50 rect2=20  | grey title="Ground Roll"')  # noise
Result('inoi','rfks inoi','SideBySideAniso')

Flow('isig','subsample100 inoi',
     '''
     add scale=1,-1 ${SOURCES[1]}
     ''')

Plot('isig','agc rect1=50 rect2=20  | grey title="After Ground Roll Attenuation"')

Result('fk_filter','subsample100 isig inoi','SideBySideAniso')


# ##############################
# LTFT for ground roll attenuation
# ##############################

# # CalculateTime-frequency using LTFT
Flow('ltft100','subsample100',
     '''
     ltft rect=20 verb=n nw=50 dw=2 niter=50
     ''')
Result('ltft100',
       '''
       math output="abs(input)" | real |
       byte allpos=y gainpanel=100 pclip=99 |
       grey3 color=j  frame1=120 frame2=7 frame3=71 label1=Time flat=n movie=2
       unit1=s label3=Offset label2="\F5 f \F-1" unit3=km
       ''')
# # Thresholding
Flow('thr100','ltft100',
     '''
     transp plane=23 memsize=1000 |
     threshold2 pclip=25 verb=y |
     transp plane=23 memsize=1000
     ''')
Result('thr100',
       '''
       math output="abs(input)" | real |
       byte allpos=y gainpanel=100 pclip=99 |
       grey3 color=j  frame1=120 frame2=7 frame3=71 label1=Time flat=n 
       unit1=s label3=Offset label2="\F5 f \F-1" unit3=km
       ''')
# # Denoise
Flow('noise100','thr100','ltft inv=y | mutter t0=-0.5 v0=0.7')
Plot('noise100',
     'agc rect1=50 rect2=20 | grey title="Ground-roll 100" unit2=km labelfat=4 titlefat=4')

Flow('signal100','subsample100 noise100','add scale=1,-1 ${SOURCES[1]}')
Plot('signal100',
     'agc rect1=50 rect2=20 | grey title="Ground-roll removal" labelfat=4 titlefat=4')
Result('sn100','mute100 signal100 noise100','SideBySideAniso')

# apply ltft to all shots
Flow('ltft','subsample',
     '''
     ltft rect=20 verb=n nw=50 dw=2 niter=50
     ''')
Flow('thr','ltft',
     '''
     transp plane=23 memsize=1000 |
     threshold2 pclip=25 verb=y |
     transp plane=23 memsize=1000
     ''')
Flow('noise','thr','ltft inv=y | mutter t0=-0.5 v0=0.7')
Flow('signal','subsample noise','add scale=1,-1 ${SOURCES[1]}')


# ##############################
# surface consitent amplitude correction
# ##############################

# Average trace amplitude
Flow('arms','signal',
     'mul $SOURCE | stack axis=1 | math output="log(input)" ')
Plot('arms','grey title="Log-Amplitude" mean=y pclip=90')

# Remove long-period offset term
Flow('arms2','arms','smooth rect1=5 | add scale=-1,1 $SOURCE')
Plot('arms2','grey title="Log-Amplitude after removing long-period offset" clip=1.13')
Result('arm','arms arms2','SideBySideAniso')

# Integer indeces for different terms
Flow('shot','arms2','math output="(x2-688)/0.05" ')
Flow('offsets','arms2','math output="(x1+3.5)/0.025" ')
Flow('receiver','arms2',
     'math output="(x1+x2-688+3.5)/0.025" ')
Flow('cmp','arms2',
     'math output="(x1/2+x2-688+3.5/2)*2/0.0.5" ')

nx = 282 # number of offsets
ns = 251 # number of shots
nt = nx*ns # number of traces

Flow('index','shot offsets receiver cmp',
     '''
     cat axis=3 ${SOURCES[1:4]} | dd type=int | 
     put n1=%d n2=4 n3=1
     ''' % nt)

# Transform from 2-D to 1-D
Flow('arms1','arms2','put n2=1 n1=%d' % nt)

prog = Program('surface-consistent.c')
sc = str(prog[0])

Flow('model',['arms1','index',sc],
     './${SOURCES[2]} index=${SOURCES[1]} verb=y')

# Least-squares inversion by conjugate gradients
Flow('sc',['arms1','index',sc,'model'],
     '''
     conjgrad ./${SOURCES[2]} index=${SOURCES[1]} 
     mod=${SOURCES[3]} niter=30
     ''')


Flow('scarms',['sc','index',sc],
     '''
     ./${SOURCES[2]} index=${SOURCES[1]} adj=n | 
     put n1=%d n2=%d
     ''' % (nx,ns))
Plot('scarms',
       '''
       grey mean=y title="Surface-Consistent Log-Amplitude" 
       clip=1.13
       ''')

Flow('adiff','arms2 scarms','add scale=1,-1 ${SOURCES[1]}')
Plot('adiff','grey title=Difference clip=1.13')
Result('adiff','scarms adiff','SideBySideAniso')


Flow('ampl','scarms',
     '''
     math output="exp(-input/2)" |
     spray axis=1 n=751 d=0.004 o=0
     ''')
Flow('ashots','signal ampl',
     'mul ${SOURCES[1]} | cut n3=1 f3=49')


Plot('signal',
       '''transp memsize=1000 plane=23 |
       byte gainpanel=each |
       grey3 frame1=500 frame2=100 frame3=120 flat=n 
       title="'Shots before Amplitude Normalization"''')

Plot('ashots',
              '''transp memsize=1000 plane=23 |
       byte gainpanel=each |
       grey3 frame1=500 frame2=100 frame3=120 flat=n
       title="'Shots after Amplitude Normalization"''')
Result('ashots','signal ashots','SideBySideAniso')

# Flow('diff',' signal ashots','add scale=1,-1 ${SOURCES[1]}')
# Result('diff',
#               '''transp memsize=1000 plane=23 |
#        byte gainpanel=each |
#        grey3 frame1=500 frame2=100 frame3=120 flat=n movie=2
#        title="'difference after surface consistent amplitude correction"''')


# ##############################
# Shot to CMPs
# ##############################

# Convert shots to CMPs
Flow('cmps','signal',
     '''
     mutter v0=3. |
     shot2cmp half=n | put o2=-1.75 d2=0.05 label2="Half-offset"
     ''')

Result('cmps',
       '''
       byte gainpanel=each | window j3=2 |
       grey3 frame1=500 frame2=36 frame3=321 flat=n
       title="CMP gathers" point1=0.7 label2=Offset label3=Midpoint
       ''')

# ##############################
# Velocity analysis and NMO
# ##############################

# Extract one CMP gather

Flow('cmp1','cmps','window n3=1 f3=300')
Plot('cmp1',' agc rect1=50 rect2=20  | grey title="CMP 300"')
Result('cmp1',' agc rect1=50 rect2=20  | grey title="CMP 300"')

# Velocity scan

Flow('vscan1','cmp1',
     '''
    vscan semblance=y v0=1.0 nv=75 dv=0.05 half=y
     ''')
Plot('vscan1',
     '''
     grey color=j allpos=y title="Semblance Scan" unit2=km/s
     ''')

mute = Program('mute.c')

Flow('vmute1','vscan1 %s' % mute[0],
     './${SOURCES[1]} t1=0.5 v1=4')

Plot('vmute1',
     '''
     grey color=j allpos=y title="Semblance Scan" unit2=km/s
     ''')

# # Automatic pick

Flow('vpick1','vmute1','pick rect1=10 rect2=20 gate=50 an=10')
Plot('vpick1',
     '''
     graph yreverse=y transp=y plotcol=7 plotfat=7 
     pad=n min2=1.4 max2=3.8 wantaxis=n wanttitle=n
     ''')

Plot('vscan2','vmute1 vpick1','Overlay')

# # NMO

Flow('nmo1','cmp1 vpick1',
     'nmo half=y  velocity=${SOURCES[1]}')
Plot('nmo1','agc rect1=50 rect2=20  | grey title="NMO CMP 300"')
Result('nmo1','agc rect1=50 rect2=20  | grey title="NMO CMP 300"')


Result('nmo1plot','cmp1 vscan2 nmo1','SideBySideAniso')


# NMO on all CMPS
v0 = 1.0
dv = 0.05
nv = 75

# Velocity scanning for all CMP gathers
Flow('scn','cmps',
     '''
     vscan semblance=y v0=%g nv=%d dv=%g half=y str=0 |
     mutter v0=0.9 t0=-4.5 inner=y
     ''' % (v0,nv,dv),split=[3,1285])

Flow('mute','scn %s' % mute[0],
     './${SOURCES[1]} t1=0.5 v1=4')

Flow('vel','mute','pick rect1=10 rect2=20 gate=50 an=10 | window')
Result('vel',
       '''
       grey title="NMO Velocity" label1="Time" label2="Lateral"
       color=j scalebar=y allpos=y bias=2.1 barlabel="Velocity"
       barreverse=y o2num=1 d2num=1 n2tic=3 labelfat=4 font=2 titlefat=4
       ''')

# NMO
Flow('nmo','cmps vel',
     '''
     nmo velocity=${SOURCES[1]} half=y
     ''')

Result('nmo',
       '''
       byte gainpanel=each | 
       grey3 frame1=500 frame2=36 frame3=642 flat=n
       title="NMOed Data" point1=0.7
       label2=Offset label3=Midpoint
       ''')

# ##############################
# (TVMF) for attenuating prestack random, spike-like noise.
# ##############################

Flow('median','nmo','tvmf nfw=6 |bandpass flo=2 | bandpass fhi=90')

#Flow('median','nmo','tvmf nfw=60 | bandpass flo=5 | bandpass fhi=90')
#Flow('median','nmoa','tvmf nfw=5')
# ##############################
# Stack
# ##############################
Flow('stack_nmo','nmo','stack')
Plot('stack_nmo',
       '''
       grey color=seismic title="NMOed data stack"
       ''')

# Median filter stack
Flow('stack_median','median','stack')
Plot('stack_median',
       '''
       grey color=seismic title="Stack after median filter"
       ''')

Result('tvmf','stack_nmo stack_median','SideBySideAniso')

# # ################
# # # Post-stack denoising using fxdecon/ spiking decon
# # ################

# # ##############################
# # FX Decon
# # ##############################

# Flow('fxdecon','stack_median','fxdecon fmin=10 fmax=90')
# Result('fxdecon',
#      'grey title="F-x Deconvolution" unit2=km labelfat=4 titlefat=4')
# #Result('fxdecon','stack_median fxdecon','SideBySideAniso')

# # Flow('decon','signal', 'fxdecon fmin=10 fmax=90 | mutter slope0=0.2')

# # # Spiking Deconvolution

# Flow('spdecon','stack_median','pef minlag=.004 maxlag=.140 pnoise=.01 mincorr=0 maxcorr=3')
# Result('spdecon',
#      'grey title="F-x Deconvolution 100" unit2=km labelfat=4 titlefat=4')
# #Result('spdecon','stack_median fxdecon','SideBySideAniso')

# # ################
# # # Kirchoff post stack time migration
# # ################
# 2D cosine transform
Flow('cosft','stack_median',
     '''
     cosft sign1=1 sign2=1 | window max1=90 | 
     mutter v0=1 half=n | put label2=Wavenumber
     ''')
Result('cosft','grey title="Cosine Transform" pclip=95')

# Ensemble of Stolt migrations with different velocities
Flow('spray','cosft',
     '''
     spray axis=3 n=90 o=1.5 d=0.06
     label=Velocity unit=km/s
     ''')
Flow('map','spray',
     'math output="sqrt(x1*x1+0.25*x3*x3*x2*x2)" ')
Result('map',
       '''
       byte gainpanel=all allpos=y | 
       grey3 title="Stolt Ensemble Map" 
       frame1=400 frame2=1000 frame3=50 color=x
       ''')

Flow('mig','spray map',
     '''
     iwarp warp=${SOURCES[1]} inv=n | pad n1=751 | 
     cosft sign1=-1 sign2=-1
     ''')
Flow('migt','mig','transp plane=23 memsize=5000')
Plot('mig',
    'grey title="Ensemble of Stolt Migrations" ',view=1)

# Migration velocity increasing with time
Flow('vmig','stack_median','math output="1.5+0.06*x1" ')
Result('vmig',
       '''
       grey color=j mean=y barreverse=y 
       title="Migration Velocity" 
       scalebar=y barlabel=Velocity barunit=km/s
       ''')

# Slice through the ensemble of migrations
Flow('slice','migt vmig','slice pick=${SOURCES[1]}')
Result('mig','slice',
      'grey title="Stolt Migration with Variable Velocity" ')
