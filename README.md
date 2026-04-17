#include <GL/glut.h>
#include <math.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#define W 1200
#define H 700
#define PI 3.14159265f

#define MAX_MISSILES  48   // increased for air-to-air combat
#define MAX_TRAIL     100
#define MAX_DEBRIS    250
#define MAX_SMOKE     150
#define NUM_STARS     85
#define NUM_CLOUDS    10
#define MAX_CIVILIANS  8
#define MAX_AIRCRAFT   4
#define MAX_WINDMILLS  2
#define MAX_TREES      12

#define GY 218

typedef enum { MS_INACTIVE=0, MS_FLYING, MS_INTERCEPTED, MS_HIT } MissileState;
typedef enum { SIDE_LEFT=0, SIDE_RIGHT=1 } Side;
// missile type: 0=attack(ground), 1=AD interceptor, 2=air-to-air
typedef enum { MT_ATTACK=0, MT_AD=1, MT_AIR=2 } MissileType;

typedef struct { float x,y,alpha; } TrailPt;

typedef struct {
    float startX,startY,endX,endY,ctrlX,ctrlY;
    float x,y,t,speed;
    MissileState state;
    Side fromSide;
    MissileType mtype;   // replaces isInterceptor bool
    TrailPt trail[MAX_TRAIL];
    int trailHead,trailCount;
    float expR,expX,expY;
    int expActive;
    int targetAircraft;  // index into aircraft[] for air-to-air missiles, -1 otherwise
} Missile;

typedef struct {
    float x,y,vx,vy,life,maxLife,r,g,b,size;
    int active;
} Debris;

typedef struct {
    float x,y,vx,vy,life,maxLife,size;
    int active;
} SmokePt;

typedef struct {
    float x,y,w,h,r,g,b;
    float damage;
    int collapsing,collapsed;
    float collapseT;
    int isLeft;
} Building;

typedef struct {
    float x,y,vx,legPhase;
    int active,dir,skin,isLeft,fleeing;
} Civilian;

typedef struct { float x,y,spd,sc,alpha; } Cloud;

typedef struct {
    float x,y,vx,rotorAngle;
    float bombTimer,bombCooldown;
    float hitTimer,vy;
    int active,isLeft,type,crashed;
    float crashX,crashY;
    // air-to-air
    float airMissileTimer, airMissileCooldown;
    int fleeing;          // set when war is over — fly off screen
} Aircraft;

typedef struct {
    float x, y, vy, health;
    int active, destroyed;
    float destroyTimer;
    float missileTimer, missileCooldown;
    int phase;
    float retreatVx;
    float retreatVy;
    float sinkDepth;
} FighterShip;

typedef struct {
    float x, y, bladeAngle;
    float spinSpeed;      // changes with nearby explosions
    int damaged;
    float damageTimer;
    int isLeft;
} Windmill;

typedef struct {
    float x, y, height, width;
    int leafType;
} Tree;

typedef struct { float x,y; Side side; int isAD; } LaunchPad;
#define MAX_PADS 8
LaunchPad pads[MAX_PADS];
int numPads=0;

typedef struct { float x,y,cx,cy; Side side; float damage; int destroyed; } TankReg;
#define MAX_TANKS_REG 2
TankReg tanks[MAX_TANKS_REG];
int numTanks=0;

Missile   missiles[MAX_MISSILES];
Debris    debris[MAX_DEBRIS];
SmokePt   smokeP[MAX_SMOKE];
Building  buildings[6];
Civilian  civs[MAX_CIVILIANS];
Cloud     clouds[NUM_CLOUDS];
Aircraft  aircraft[MAX_AIRCRAFT];
FighterShip ship;
Windmill  windmills[MAX_WINDMILLS];
Tree      trees[MAX_TREES];

float starX[NUM_STARS],starY[NUM_STARS],starPhase[NUM_STARS],starSz[NUM_STARS];
float globalTime=0.f,gameSpeed=1.f;
int   paused=0,leftScore=0,rightScore=0;
float leftMissileTimer=0,rightMissileTimer=0;
float leftMissileDelay=4.5f,rightMissileDelay=4.2f;
float tankTimerL=0,tankTimerR=0,tankDelayL=9.f,tankDelayR=10.f;

int allBuildingsDestroyed = 0;
int warJustEnded = 0;   // one-shot flag to clear missiles cleanly

float randF(float a,float b){return a+(b-a)*(float)rand()/RAND_MAX;}
int   randI(int a,int b){return a+rand()%(b-a+1);}
float clampF(float v,float lo,float hi){return v<lo?lo:v>hi?hi:v;}

void lineDDA(float x1,float y1,float x2,float y2){
    float dx=x2-x1,dy=y2-y1,st=fabsf(dx)>fabsf(dy)?fabsf(dx):fabsf(dy);
    if(st<1)return;
    float xi=dx/st,yi=dy/st,x=x1,y=y1;
    glBegin(GL_POINTS);
    for(int i=0;i<=(int)st;i++){glVertex2f(roundf(x),roundf(y));x+=xi;y+=yi;}
    glEnd();
}
void lineB(int x1,int y1,int x2,int y2){
    int dx=abs(x2-x1),dy=abs(y2-y1),sx=x1<x2?1:-1,sy=y1<y2?1:-1,e=dx-dy;
    glBegin(GL_POINTS);
    for(;;){glVertex2i(x1,y1);if(x1==x2&&y1==y2)break;
        int e2=2*e;if(e2>-dy){e-=dy;x1+=sx;}if(e2<dx){e+=dx;y1+=sy;}}
    glEnd();
}
void circleMid(float xc,float yc,float r){
    int xi=0,yi=(int)r,p=1-(int)r;
    glBegin(GL_POINTS);
    while(xi<=yi){
        glVertex2f(xc+xi,yc+yi);glVertex2f(xc-xi,yc+yi);
        glVertex2f(xc+xi,yc-yi);glVertex2f(xc-xi,yc-yi);
        glVertex2f(xc+yi,yc+xi);glVertex2f(xc-yi,yc+xi);
        glVertex2f(xc+yi,yc-xi);glVertex2f(xc-yi,yc-xi);
        xi++;p=(p<0)?p+2*xi+1:(yi--,p+2*(xi-yi)+1);}
    glEnd();
}
void fillCircle(float x,float y,float r,int s){
    glBegin(GL_TRIANGLE_FAN);glVertex2f(x,y);
    for(int i=0;i<=s;i++){float a=2*PI*i/s;glVertex2f(x+r*cosf(a),y+r*sinf(a));}
    glEnd();
}
void fillEllipse(float x,float y,float rx,float ry,int s){
    glBegin(GL_TRIANGLE_FAN);glVertex2f(x,y);
    for(int i=0;i<=s;i++){float a=2*PI*i/s;glVertex2f(x+rx*cosf(a),y+ry*sinf(a));}
    glEnd();
}
void bezierPt(float t,float x0,float y0,float cx,float cy,
              float x1,float y1,float*bx,float*by){
    float u=1-t;*bx=u*u*x0+2*u*t*cx+t*t*x1;*by=u*u*y0+2*u*t*cy+t*t*y1;
}
void thickLine(float x1,float y1,float x2,float y2,float th){
    float dx=x2-x1,dy=y2-y1,len=sqrtf(dx*dx+dy*dy);
    if(len<.001f)return;
    float px=-dy/len*th*.5f,py=dx/len*th*.5f;
    glBegin(GL_QUADS);
    glVertex2f(x1+px,y1+py);glVertex2f(x1-px,y1-py);
    glVertex2f(x2-px,y2-py);glVertex2f(x2+px,y2+py);
    glEnd();
}

void spawnDebris(float x,float y,int n,float r,float g,float b){
    for(int i=0;i<n;i++) for(int j=0;j<MAX_DEBRIS;j++) if(!debris[j].active){
        float a=randF(0,2*PI),sp=randF(2.f,6.f);
        debris[j].x=x;debris[j].y=y;
        debris[j].vx=cosf(a)*sp;debris[j].vy=sinf(a)*sp+randF(0,3.f);
        debris[j].life=randF(30,90);debris[j].maxLife=90;
        debris[j].r=r;debris[j].g=g;debris[j].b=b;
        debris[j].size=randF(2,5);debris[j].active=1;break;
    }
}
void spawnSmoke(float x,float y,int n){
    for(int i=0;i<n;i++) for(int j=0;j<MAX_SMOKE;j++) if(!smokeP[j].active){
        smokeP[j].x=x+randF(-9,9);smokeP[j].y=y+randF(-5,5);
        smokeP[j].vx=randF(-.4f,.4f);smokeP[j].vy=randF(.5f,1.8f);
        smokeP[j].life=randF(50,110);smokeP[j].maxLife=110;
        smokeP[j].size=randF(6,17);smokeP[j].active=1;break;
    }
}
void updateParticles(){
    for(int i=0;i<MAX_DEBRIS;i++) if(debris[i].active){
        debris[i].x+=debris[i].vx;debris[i].y+=debris[i].vy;
        debris[i].vy-=.14f;debris[i].life--;
        if(debris[i].life<=0)debris[i].active=0;}
    for(int i=0;i<MAX_SMOKE;i++) if(smokeP[i].active){
        smokeP[i].x+=smokeP[i].vx;smokeP[i].y+=smokeP[i].vy;
        smokeP[i].size+=.13f;smokeP[i].life--;
        if(smokeP[i].life<=0)smokeP[i].active=0;}
}
void drawParticles(){
    for(int i=0;i<MAX_DEBRIS;i++) if(debris[i].active){
        float a=debris[i].life/debris[i].maxLife;
        glColor4f(debris[i].r,debris[i].g,debris[i].b,a);
        fillCircle(debris[i].x,debris[i].y,debris[i].size*a+1,6);}
    for(int i=0;i<MAX_SMOKE;i++) if(smokeP[i].active){
        float a=(smokeP[i].life/smokeP[i].maxLife)*.26f;
        glColor4f(.42f,.40f,.40f,a);
        fillCircle(smokeP[i].x,smokeP[i].y,smokeP[i].size,10);}
}

void initBuildings(){
    buildings[0]=(Building){30, GY+6,100,168,.52f,.38f,.18f,0,0,0,0,1};
    buildings[1]=(Building){148,GY+6, 88,155,.48f,.35f,.16f,0,0,0,0,1};
    buildings[2]=(Building){250,GY+6, 78,150,.46f,.33f,.15f,0,0,0,0,1};
    buildings[3]=(Building){872,GY+6, 78,150,.62f,.64f,.66f,0,0,0,0,0};
    buildings[4]=(Building){964,GY+6, 88,155,.60f,.62f,.64f,0,0,0,0,0};
    buildings[5]=(Building){1068,GY+6,100,168,.57f,.59f,.62f,0,0,0,0,0};
}

void drawRubble(Building*b){
    float cx=b->x+b->w/2,gy=b->y;
    glColor3f(.28f,.24f,.16f);fillEllipse(cx,gy,b->w*.60f,10,16);
    for(int i=0;i<8;i++){
        float rx=b->x+4+i*(b->w/8.f),rh=4+((i*7)%9);
        float rc=(i%3==0)?.26f:(i%3==1)?.33f:.20f;
        glColor3f(rc,rc*.78f,rc*.58f);
        glBegin(GL_QUADS);
        glVertex2f(rx,gy);glVertex2f(rx+9,gy);
        glVertex2f(rx+7,gy+rh);glVertex2f(rx+2,gy+rh);
        glEnd();
    }
    glColor4f(.46f,.42f,.34f,.12f);fillEllipse(cx,gy+12,b->w*.62f,15,16);
}

void drawBuilding(Building*b){
    if(b->collapsed){drawRubble(b);return;}
    float bh=b->h,ox=b->x,oy=b->y;
    if(b->collapsing){
        bh=b->h*(1.f-clampF(b->collapseT,0,1));
        if(bh<5.f){b->collapsing=0;b->collapsed=1;return;}
    }
    float dr=clampF(b->damage*.30f,0,.30f);
    glColor4f(0,0,0,.17f);fillEllipse(ox+b->w/2,oy+2,b->w*.46f,5,14);
    glColor3f(b->r-dr,b->g-dr,b->b-dr);
    glBegin(GL_QUADS);
    glVertex2f(ox,oy);glVertex2f(ox+b->w,oy);
    glVertex2f(ox+b->w,oy+bh);glVertex2f(ox,oy+bh);
    glEnd();
    glColor4f(0,0,0,.09f);
    int fl=(int)(bh/36);for(int f=1;f<=fl;f++)lineDDA(ox,oy+f*36,ox+b->w,oy+f*36);
    int cols=(int)((b->w-12)/26),rows=(int)((bh-12)/32);
    for(int c=0;c<cols;c++) for(int r2=0;r2<rows;r2++){
        float wx=ox+6+c*26,wy=oy+7+r2*32;
        if(wy+16>oy+bh)continue;
        int dark=((c*5+r2*3)%4==0)||(b->damage>.3f&&rand()%200<(int)(b->damage*50));
        if(dark)glColor4f(.07f,.07f,.10f,1.f);
        else if(b->isLeft)glColor4f(.96f,.78f,.22f,.90f);
        else glColor4f(.38f,.80f,1.f,.86f);
        glBegin(GL_QUADS);
        glVertex2f(wx,wy);glVertex2f(wx+18,wy);
        glVertex2f(wx+18,wy+15);glVertex2f(wx,wy+15);
        glEnd();
        if(!dark){glColor4f(1,1,1,.13f);
            glBegin(GL_TRIANGLES);
            glVertex2f(wx,wy+15);glVertex2f(wx,wy);glVertex2f(wx+7,wy+15);glEnd();}
        glColor4f(.16f,.16f,.20f,.65f);
        lineDDA(wx,wy,wx+18,wy);lineDDA(wx+18,wy,wx+18,wy+15);
        lineDDA(wx+18,wy+15,wx,wy+15);lineDDA(wx,wy+15,wx,wy);
    }
    glColor3f(b->r*.62f,b->g*.62f,b->b*.62f);
    glBegin(GL_QUADS);
    glVertex2f(ox-3,oy+bh);glVertex2f(ox+b->w+3,oy+bh);
    glVertex2f(ox+b->w+3,oy+bh+6);glVertex2f(ox-3,oy+bh+6);
    glEnd();
    if(b->damage>.25f){
        glColor4f(.03f,.02f,.01f,b->damage*.56f);
        fillCircle(ox+b->w*.64f,oy+bh*.68f,18*b->damage,14);
        fillCircle(ox+b->w*.28f,oy+bh*.46f,12*b->damage,12);}
    if(b->damage>.5f){
        glColor4f(.08f,.06f,.03f,b->damage*.65f);
        lineDDA(ox+b->w*.58f,oy+bh*.35f,ox+b->w*.68f,oy+bh*.76f);
        lineDDA(ox+b->w*.60f,oy+bh*.44f,ox+b->w*.52f,oy+bh*.70f);}
    if(b->damage<.44f){
        glColor3f(.44f,.44f,.46f);
        thickLine(ox+b->w/2,oy+bh+6,ox+b->w/2,oy+bh+28,2.f);
        glColor4f(.90f,.06f,.06f,1.f);fillCircle(ox+b->w/2,oy+bh+30,2.8f,10);}
}

void damageBuilding(int idx,float amt){
    if(idx<0||idx>=6||buildings[idx].collapsed||buildings[idx].collapsing)return;
    buildings[idx].damage+=amt;
    if(buildings[idx].damage>=.85f){
        buildings[idx].collapsing=1;buildings[idx].collapseT=0;
    }
}

void checkAllBuildingsDestroyed(){
    if(allBuildingsDestroyed) return;
    int count = 0;
    for(int i=3;i<6;i++){
        if(buildings[i].collapsed) count++;
    }
    if(count >= 3){
        allBuildingsDestroyed = 1;
        warJustEnded = 1;
    }
}

void updateBuildings(){
    for(int i=0;i<6;i++) if(buildings[i].collapsing){
        buildings[i].collapseT+=.006f*gameSpeed;
        float cx=buildings[i].x+buildings[i].w/2;
        float cy=buildings[i].y+buildings[i].h*(1-buildings[i].collapseT);
        spawnSmoke(cx,cy,2);
        if(rand()%3==0)spawnDebris(cx+randF(-26,26),cy,3,.34f,.26f,.12f);}

    checkAllBuildingsDestroyed();
}

void drawTankAt(float cx,float cy,int facingRight,float sc,
                float dmg,int destroyed){
    float d=facingRight?1.f:-1.f;
    if(destroyed){
        glColor3f(.16f,.14f,.10f);
        fillEllipse(cx,cy,36*sc,8*sc,14);
        glColor3f(.22f,.18f,.12f);
        glBegin(GL_QUADS);
        glVertex2f(cx-20*sc,cy+5*sc);glVertex2f(cx+20*sc,cy+5*sc);
        glVertex2f(cx+18*sc,cy+16*sc);glVertex2f(cx-18*sc,cy+16*sc);
        glEnd();
        glColor4f(.08f,.06f,.04f,.80f);fillCircle(cx,cy+12*sc,14*sc,12);
        float ff=.5f+.5f*sinf(globalTime*8.f);
        glColor4f(1.f,.35f,.0f,ff*.70f);fillCircle(cx,cy+18*sc,10*sc*ff,10);
        glColor4f(1.f,.70f,.0f,ff*.55f);fillCircle(cx,cy+22*sc,6*sc*ff,8);
        return;
    }
    glColor4f(0,0,0,.19f);fillEllipse(cx,cy-2,36*sc,5*sc,12);
    glColor3f(.13f,.13f,.10f);
    fillEllipse(cx-11*sc,cy,8*sc,5*sc,10);
    fillEllipse(cx+11*sc,cy,8*sc,5*sc,10);
    glBegin(GL_QUADS);
    glVertex2f(cx-11*sc,cy-5*sc);glVertex2f(cx+11*sc,cy-5*sc);
    glVertex2f(cx+11*sc,cy+5*sc);glVertex2f(cx-11*sc,cy+5*sc);
    glEnd();
    glColor3f(.20f,.20f,.16f);
    for(int k=0;k<5;k++){float tx=cx-14*sc+k*6*sc;
        glBegin(GL_QUADS);
        glVertex2f(tx,cy-5.5f*sc);glVertex2f(tx+4.5f*sc,cy-5.5f*sc);
        glVertex2f(tx+4.5f*sc,cy+5.5f*sc);glVertex2f(tx,cy+5.5f*sc);
        glEnd();}
    for(int k=0;k<4;k++){float wx=cx-9*sc+k*6.5f*sc;
        glColor3f(.16f,.16f,.13f);fillCircle(wx,cy,3*sc,10);
        glColor3f(.24f,.24f,.20f);fillCircle(wx,cy,1.8f*sc,8);}
    float hdr=clampF(dmg*.3f,0,.3f);
    glColor3f(.26f-hdr,.31f-hdr,.17f-hdr);
    glBegin(GL_QUADS);
    glVertex2f(cx-17*sc,cy+5*sc);glVertex2f(cx+17*sc,cy+5*sc);
    glVertex2f(cx+15*sc,cy+16*sc);glVertex2f(cx-15*sc,cy+16*sc);
    glEnd();
    glColor3f(.30f-hdr,.36f-hdr,.19f-hdr);
    glBegin(GL_QUADS);
    glVertex2f(cx-15*sc,cy+15*sc);glVertex2f(cx+15*sc,cy+15*sc);
    glVertex2f(cx+11*sc,cy+23*sc);glVertex2f(cx-11*sc,cy+23*sc);
    glEnd();
    glColor3f(.22f,.27f,.15f);fillEllipse(cx,cy+23*sc,11*sc,7*sc,16);
    glColor3f(.27f,.33f,.18f);fillEllipse(cx,cy+27*sc,9.5f*sc,7.5f*sc,16);
    glColor3f(.19f,.23f,.12f);fillCircle(cx+2*sc,cy+32*sc,3.2f*sc,10);
    glColor3f(.17f,.21f,.11f);
    thickLine(cx+d*9*sc,cy+27*sc,cx+d*38*sc,cy+25*sc,3.2f*sc);
    glColor3f(.12f,.15f,.08f);
    fillCircle(cx+d*38*sc,cy+25*sc,1.8f*sc,8);
    glColor4f(.15f,.19f,.06f,.46f);
    fillCircle(cx-5*sc,cy+11*sc,3.8f*sc,8);
    fillCircle(cx+7*sc,cy+13*sc,3.2f*sc,8);
    if(dmg>.3f){glColor4f(.04f,.03f,.02f,dmg*.5f);
        fillCircle(cx,cy+18*sc,8*sc*dmg,10);}
}

void registerTank(float cx,float cy,float bx2,float by2,Side side){
    if(numTanks<MAX_TANKS_REG){
        tanks[numTanks].x=cx;tanks[numTanks].y=cy;
        tanks[numTanks].cx=bx2;tanks[numTanks].cy=by2;
        tanks[numTanks].side=side;
        tanks[numTanks].damage=0;tanks[numTanks].destroyed=0;
        numTanks++;
    }
}

void drawAttackLauncher(float x,float y,float ang,Side side,float sc){
    int dir=(side==SIDE_LEFT)?1:-1;
    float bx1=x+28*sc,by1=y+20*sc;
    float tipX=bx1+dir*42*sc*cosf(ang);
    float tipY=by1+42*sc*sinf(ang);
    if(numPads<MAX_PADS){
        pads[numPads].x=tipX;pads[numPads].y=tipY;
        pads[numPads].side=side;pads[numPads].isAD=0;
        numPads++;
    }
    glColor3f(.10f,.10f,.10f);
    fillCircle(x+8*sc,y,7*sc,12);fillCircle(x+48*sc,y,7*sc,12);
    glColor3f(.20f,.20f,.20f);
    circleMid(x+8*sc,y,7*sc);circleMid(x+48*sc,y,7*sc);
    glColor3f(.15f,.20f,.08f);
    glBegin(GL_QUADS);
    glVertex2f(x,y);glVertex2f(x+56*sc,y);
    glVertex2f(x+56*sc,y+9*sc);glVertex2f(x,y+9*sc);
    glEnd();
    glColor3f(.18f,.24f,.10f);
    glBegin(GL_QUADS);
    glVertex2f(x+4*sc,y+9*sc);glVertex2f(x+52*sc,y+9*sc);
    glVertex2f(x+52*sc,y+23*sc);glVertex2f(x+4*sc,y+23*sc);
    glEnd();
    glColor3f(.22f,.28f,.12f);thickLine(bx1,by1,tipX,tipY,4.5f*sc);
    glColor4f(.10f,.16f,.05f,.46f);
    fillCircle(x+14*sc,y+16*sc,4*sc,8);fillCircle(x+38*sc,y+14*sc,3.5f*sc,8);
    glColor3f(.56f,.48f,.13f);
    fillEllipse(bx1+dir*14*sc,by1+10*sc*sinf(ang)*.5f,1.9f*sc,7*sc,10);
}

void drawADLauncher(float x,float y,Side side){
    float tipX=x, tipY=y+55;
    if(numPads<MAX_PADS){
        pads[numPads].x=tipX;pads[numPads].y=tipY;
        pads[numPads].side=side;pads[numPads].isAD=1;
        numPads++;
    }
    glColor3f(.18f,.22f,.12f);
    glBegin(GL_QUADS);
    glVertex2f(x-18,y);glVertex2f(x+18,y);
    glVertex2f(x+16,y+14);glVertex2f(x-16,y+14);
    glEnd();
    glColor3f(.22f,.28f,.14f);
    fillEllipse(x,y+14,12,6,14);
    glColor3f(.28f,.34f,.16f);
    thickLine(x-7,y+14,x-7,y+55,5);
    thickLine(x+7,y+14,x+7,y+55,5);
    glColor3f(.62f,.56f,.18f);
    fillEllipse(x-7,y+42,3,12,10);
    glColor3f(.75f,.68f,.24f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x-7,y+56);glVertex2f(x-11,y+44);glVertex2f(x-3,y+44);
    glEnd();
    glColor3f(.62f,.56f,.18f);
    fillEllipse(x+7,y+40,3,12,10);
    glColor3f(.75f,.68f,.24f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x+7,y+54);glVertex2f(x+3,y+42);glVertex2f(x+11,y+42);
    glEnd();
    glColor3f(.42f,.42f,.44f);
    thickLine(x+14,y+8,x+28,y+22,2);
    glBegin(GL_TRIANGLE_FAN);
    glVertex2f(x+28,y+22);
    for(int i=0;i<=12;i++){float a=PI+PI*i/12;
        glVertex2f(x+28+14*cosf(a),y+22+8*sinf(a));}
    glEnd();
    float ra=globalTime*1.8f;
    glColor3f(.50f,.50f,.52f);
    lineDDA(x+28,y+16,x+28+8*cosf(ra),y+16+8*sinf(ra));
}

void initMissileS(Missile*m,float sx,float sy,float ex,float ey,
                  float cx2,float cy2,float spd,Side from,MissileType mt){
    m->startX=sx;m->startY=sy;m->endX=ex;m->endY=ey;
    m->ctrlX=cx2;m->ctrlY=cy2;m->x=sx;m->y=sy;
    m->t=0;m->speed=spd;m->state=MS_FLYING;
    m->fromSide=from;m->mtype=mt;
    m->trailHead=0;m->trailCount=0;m->expR=0;m->expActive=0;
    m->targetAircraft=-1;
}
int freeMissile(){for(int i=0;i<MAX_MISSILES;i++)if(missiles[i].state==MS_INACTIVE)return i;return-1;}

int getPad(Side side,int isAD){
    int idx[MAX_PADS],cnt=0;
    for(int i=0;i<numPads;i++)
        if(pads[i].side==side&&pads[i].isAD==isAD)idx[cnt++]=i;
    if(!cnt)return -1;
    return idx[randI(0,cnt-1)];
}

void launchAttackMissile(Side from){
    if(allBuildingsDestroyed) return;
    int idx=freeMissile();if(idx<0)return;
    int pidx=getPad(from,0);if(pidx<0)return;
    int t=(from==SIDE_LEFT)?(3+randI(0,2)):randI(0,2);
    if(buildings[t].collapsed)return;
    float tx=buildings[t].x+buildings[t].w/2;
    float ty=buildings[t].y+buildings[t].h*.7f;
    float sx=pads[pidx].x,sy=pads[pidx].y;
    float cx2=(sx+tx)/2+randF(-55,55),cy2=sy+randF(270,370);
    initMissileS(&missiles[idx],sx,sy,tx,ty,cx2,cy2,randF(.0012f,.0019f),from,MT_ATTACK);
}

void launchShipMissile(){
    if(!ship.active || ship.destroyed) return;
    if(allBuildingsDestroyed) return;

    int idx=freeMissile();
    if(idx<0) return;

    int validBuildings[3];
    int validCount = 0;
    for(int i=3; i<6; i++){
        if(!buildings[i].collapsed && !buildings[i].collapsing){
            validBuildings[validCount++] = i;
        }
    }
    if(validCount == 0) return;

    int targetIdx = validBuildings[randI(0, validCount-1)];
    float tx = buildings[targetIdx].x + buildings[targetIdx].w/2;
    float ty = buildings[targetIdx].y + buildings[targetIdx].h*.7f;

    float sx = ship.x;
    float sy = ship.y + 20;

    float cx2 = (sx + tx)/2 + randF(-40, 40);
    float cy2 = (sy + ty)/2 + randF(80, 150);

    initMissileS(&missiles[idx], sx, sy, tx, ty, cx2, cy2, randF(.0018f, .0025f), SIDE_LEFT, MT_ATTACK);
    spawnSmoke(sx, sy, 3);
}

void launchTankShell(Side from){
    if(allBuildingsDestroyed) return;
    int idx=freeMissile();if(idx<0)return;
    float sx=0,sy=0;int found=0;
    for(int i=0;i<numTanks;i++)
        if(tanks[i].side==from&&!tanks[i].destroyed){sx=tanks[i].cx;sy=tanks[i].cy;found=1;break;}
    if(!found)return;
    int t=(from==SIDE_LEFT)?3:2;
    if(buildings[t].collapsed){t=(from==SIDE_LEFT)?4:1;}
    if(buildings[t].collapsed)return;
    float tx=buildings[t].x+buildings[t].w/2;
    float ty=buildings[t].y+buildings[t].h*.5f;
    float cx2=(sx+tx)/2+randF(-30,30),cy2=sy+randF(180,280);
    initMissileS(&missiles[idx],sx,sy,tx,ty,cx2,cy2,randF(.0022f,.0030f),from,MT_ATTACK);
    spawnDebris(sx,sy,6,1.f,.8f,.2f);spawnSmoke(sx,sy,3);
}

void launchADIntercept(Side defSide,float tx,float ty){
    if(allBuildingsDestroyed) return;
    int idx=freeMissile();if(idx<0)return;
    int pidx=getPad(defSide,1);if(pidx<0)return;
    float sx=pads[pidx].x,sy=pads[pidx].y;
    float cx2=(sx+tx)/2,cy2=ty+randF(80,140);
    initMissileS(&missiles[idx],sx,sy,tx,ty,cx2,cy2,.0068f,defSide,MT_AD);
}

// -----------------------------------------------------------------------
// launchAirToAirMissile: aircraft fires at enemy aircraft
// -----------------------------------------------------------------------
void launchAirToAirMissile(int shooterIdx, int targetIdx){
    int mi=freeMissile(); if(mi<0) return;
    Aircraft*shooter=&aircraft[shooterIdx];
    Aircraft*target=&aircraft[targetIdx];

    float sx=shooter->x, sy=shooter->y;
    float tx=target->x,  ty=target->y;
    float cx=(sx+tx)/2 + randF(-40,40);
    float cy=(sy+ty)/2 + randF(-30,60);

    Side side = shooter->isLeft ? SIDE_LEFT : SIDE_RIGHT;
    initMissileS(&missiles[mi], sx, sy, tx, ty, cx, cy, randF(.0045f,.0060f), side, MT_AIR);
    missiles[mi].targetAircraft = targetIdx;
    spawnSmoke(sx, sy, 2);
}

// -----------------------------------------------------------------------
// FIX: checkIntercept now properly only intercepts ENEMY attack missiles
// AD missiles intercept ATTACK missiles from the opposite side
// -----------------------------------------------------------------------
void checkIntercept(){
    for(int i=0;i<MAX_MISSILES;i++){
        if(missiles[i].state!=MS_FLYING) continue;
        if(missiles[i].mtype!=MT_ATTACK) continue;  // only attack missiles get intercepted

        for(int j=0;j<MAX_MISSILES;j++){
            if(i==j) continue;
            if(missiles[j].state!=MS_FLYING) continue;
            if(missiles[j].mtype!=MT_AD) continue;  // only AD missiles intercept

            // AD must be from OPPOSITE side of the attack missile
            if(missiles[j].fromSide == missiles[i].fromSide) continue;

            float dx=missiles[i].x-missiles[j].x, dy=missiles[i].y-missiles[j].y;
            if(sqrtf(dx*dx+dy*dy)<26.f){
                float ex2=(missiles[i].x+missiles[j].x)/2;
                float ey2=(missiles[i].y+missiles[j].y)/2;
                for(int k=0;k<2;k++){
                    Missile*m=(k==0)?&missiles[i]:&missiles[j];
                    m->state=MS_INTERCEPTED;m->expX=ex2;m->expY=ey2;m->expR=2;m->expActive=1;
                }
                spawnDebris(ex2,ey2,18,1.f,.5f,.1f);spawnSmoke(ex2,ey2,8);
            }
        }
    }
}

// -----------------------------------------------------------------------
// checkADvsAircraft: AD missiles hit enemy aircraft
// -----------------------------------------------------------------------
void checkADvsAircraft(){
    for(int j=0;j<MAX_MISSILES;j++){
        if(missiles[j].state!=MS_FLYING || missiles[j].mtype!=MT_AD) continue;
        for(int a=0;a<MAX_AIRCRAFT;a++){
            if(!aircraft[a].active||aircraft[a].crashed) continue;
            if(aircraft[a].fleeing) continue;
            int acLeft=aircraft[a].isLeft;
            int adLeft=(missiles[j].fromSide==SIDE_LEFT);
            if(acLeft==adLeft) continue;  // same side, don't shoot own aircraft
            float dx=missiles[j].x-aircraft[a].x;
            float dy=missiles[j].y-aircraft[a].y;
            if(sqrtf(dx*dx+dy*dy)<32.f){
                aircraft[a].hitTimer=1.f;
                aircraft[a].vy=-0.6f;
                float ex2=aircraft[a].x,ey2=aircraft[a].y;
                missiles[j].state=MS_INTERCEPTED;
                missiles[j].expX=ex2;missiles[j].expY=ey2;
                missiles[j].expR=2;missiles[j].expActive=1;
                spawnDebris(ex2,ey2,22,1.f,.45f,.05f);
                spawnSmoke(ex2,ey2,10);
            }
        }
    }
}

// -----------------------------------------------------------------------
// checkAirToAirHits: air-to-air missiles hit target aircraft
// -----------------------------------------------------------------------
void checkAirToAirHits(){
    for(int j=0;j<MAX_MISSILES;j++){
        if(missiles[j].state!=MS_FLYING || missiles[j].mtype!=MT_AIR) continue;
        int tgt = missiles[j].targetAircraft;
        if(tgt<0||tgt>=MAX_AIRCRAFT) continue;
        if(!aircraft[tgt].active||aircraft[tgt].crashed) continue;

        float dx=missiles[j].x-aircraft[tgt].x;
        float dy=missiles[j].y-aircraft[tgt].y;
        if(sqrtf(dx*dx+dy*dy)<38.f || missiles[j].t>=1.f){
            aircraft[tgt].hitTimer=1.2f;
            aircraft[tgt].vy=-0.8f;
            float ex2=aircraft[tgt].x, ey2=aircraft[tgt].y;
            missiles[j].state=MS_INTERCEPTED;
            missiles[j].expX=ex2;missiles[j].expY=ey2;
            missiles[j].expR=4;missiles[j].expActive=1;
            spawnDebris(ex2,ey2,30,1.f,.4f,.05f);
            spawnSmoke(ex2,ey2,14);
        }
    }
}

// -----------------------------------------------------------------------
// clearAllAttackMissiles: called once when war ends to remove in-flight
// attack missiles instantly (no more buildings should be hit after war end)
// -----------------------------------------------------------------------
void clearAllAttackMissiles(){
    for(int i=0;i<MAX_MISSILES;i++){
        if(missiles[i].state==MS_FLYING && missiles[i].mtype==MT_ATTACK){
            // trigger a small explosion in-air then deactivate
            missiles[i].state=MS_INTERCEPTED;
            missiles[i].expX=missiles[i].x;
            missiles[i].expY=missiles[i].y;
            missiles[i].expR=4;
            missiles[i].expActive=1;
            spawnSmoke(missiles[i].x, missiles[i].y, 3);
        }
    }
}

void addTrail(Missile*m){
    m->trail[m->trailHead].x=m->x;m->trail[m->trailHead].y=m->y;
    m->trail[m->trailHead].alpha=.55f;
    m->trailHead=(m->trailHead+1)%MAX_TRAIL;
    if(m->trailCount<MAX_TRAIL)m->trailCount++;
}

void updateMissiles(){
    for(int i=0;i<MAX_MISSILES;i++){
        Missile*m=&missiles[i];
        if(m->state==MS_FLYING){
            m->t+=m->speed*gameSpeed;
            float pt=m->t-m->speed*gameSpeed;if(pt<0)pt=0;
            float px,py;
            bezierPt(pt,m->startX,m->startY,m->ctrlX,m->ctrlY,m->endX,m->endY,&px,&py);
            bezierPt(m->t,m->startX,m->startY,m->ctrlX,m->ctrlY,m->endX,m->endY,&m->x,&m->y);
            addTrail(m);

            // AD interception trigger: only for attack missiles from opposite side
            if(m->mtype==MT_ATTACK && m->t>.22f && m->t<.60f && rand()%2200<(int)(gameSpeed*9)){
                Side def=(m->fromSide==SIDE_LEFT)?SIDE_RIGHT:SIDE_LEFT;
                launchADIntercept(def,m->x,m->y);
            }

            if(m->t>=1.f){
                m->state=MS_HIT;m->expX=m->endX;m->expY=m->endY;
                m->expR=2;m->expActive=1;
                if(m->mtype==MT_AIR){
                    // already handled in checkAirToAirHits
                    m->expActive=1;
                    spawnDebris(m->expX,m->expY,18,1.f,.5f,.1f);
                } else {
                    spawnDebris(m->expX,m->expY,28,1.f,.32f,.02f);
                    spawnSmoke(m->expX,m->expY,13);
                }
                if(m->mtype==MT_ATTACK && !allBuildingsDestroyed){
                    float best=9999;int bi=-1;
                    for(int b=0;b<6;b++){
                        if(buildings[b].collapsed)continue;
                        float d=fabsf(buildings[b].x+buildings[b].w/2-m->endX);
                        if(d<best){best=d;bi=b;}
                    }
                    if(bi>=0)damageBuilding(bi,randF(.24f,.40f));
                    for(int t2=0;t2<numTanks;t2++){
                        float d=sqrtf((tanks[t2].x-m->endX)*(tanks[t2].x-m->endX)+
                                      (tanks[t2].y-m->endY)*(tanks[t2].y-m->endY));
                        if(d<60&&!tanks[t2].destroyed){
                            tanks[t2].damage+=randF(.3f,.6f);
                            if(tanks[t2].damage>=1.f){
                                tanks[t2].destroyed=1;
                                spawnDebris(tanks[t2].x,tanks[t2].y+15,30,.18f,.14f,.08f);
                            }
                        }
                    }
                }
            }
        }
        if(m->expActive){
            m->expR+=1.9f*gameSpeed;
            float maxR=(m->state==MS_INTERCEPTED)?38:58;
            if(m->mtype==MT_AIR) maxR=48;
            if(m->expR>maxR){m->expActive=0;if(m->state!=MS_FLYING)m->state=MS_INACTIVE;}
        }
    }
    checkIntercept();
    checkADvsAircraft();
    checkAirToAirHits();
}

void drawMissileShape(float x,float y,float dx,float dy,
                      float R,float G,float B,int isAD){
    float len=sqrtf(dx*dx+dy*dy);if(len<.001f)return;
    float ux=dx/len,uy=dy/len,px=-uy,py=ux;
    float bL=isAD?15.f:22.f,bW=isAD?2.2f:3.5f;
    glColor3f(R,G,B);
    glBegin(GL_QUADS);
    glVertex2f(x+px*bW-ux*2,y+py*bW-uy*2);
    glVertex2f(x-px*bW-ux*2,y-py*bW-uy*2);
    glVertex2f(x-px*bW-ux*(bL+2),y-py*bW-uy*(bL+2));
    glVertex2f(x+px*bW-ux*(bL+2),y+py*bW-uy*(bL+2));
    glEnd();
    glColor3f(R*1.1f,G*1.05f,B*.70f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x+ux*15,y+uy*15);
    glVertex2f(x+px*bW,y+py*bW);glVertex2f(x-px*bW,y-py*bW);
    glEnd();
    glColor3f(R*.55f,G*.55f,B*.55f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x-ux*(bL-2),y-uy*(bL-2));
    glVertex2f(x+px*6-ux*(bL+5),y+py*6-uy*(bL+5));
    glVertex2f(x-ux*(bL+7),y-uy*(bL+7));
    glVertex2f(x-ux*(bL-2),y-uy*(bL-2));
    glVertex2f(x-px*6-ux*(bL+5),y-py*6-uy*(bL+5));
    glVertex2f(x-ux*(bL+7),y-uy*(bL+7));
    glEnd();
    if(isAD==1){glColor4f(.4f,.8f,1.f,.85f);}
    else if(isAD==2){glColor4f(.2f,1.f,.4f,.90f);}  // air-to-air: green
    else{glColor4f(1.f,.44f,.06f,.86f);}
    glBegin(GL_TRIANGLES);
    glVertex2f(x+px*3-ux*(bL+2),y+py*3-uy*(bL+2));
    glVertex2f(x-px*3-ux*(bL+2),y-py*3-uy*(bL+2));
    glVertex2f(x-ux*(bL+13),y-uy*(bL+13));
    glEnd();
    glColor4f(1.f,.88f,.50f,.72f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x+px*1.5f-ux*(bL+2),y+py*1.5f-uy*(bL+2));
    glVertex2f(x-px*1.5f-ux*(bL+2),y-py*1.5f-uy*(bL+2));
    glVertex2f(x-ux*(bL+7),y-uy*(bL+7));
    glEnd();
}

void drawMissileTrail(Missile*m){
    int st=(m->trailHead-m->trailCount+MAX_TRAIL)%MAX_TRAIL;
    for(int i=0;i<m->trailCount;i++){
        int idx=(st+i)%MAX_TRAIL;
        float fade=(float)i/m->trailCount;
        float al=m->trail[idx].alpha*fade*.38f;
        if(m->mtype==MT_AD)glColor4f(.25f,.55f,1.f,al);
        else if(m->mtype==MT_AIR)glColor4f(.15f,.85f,.35f,al);
        else glColor4f(.65f,.62f,.60f,al);
        fillCircle(m->trail[idx].x,m->trail[idx].y,2+fade*3.2f,7);
    }
}

void drawExplosion(float x,float y,float r,int intercept){
    glColor4f(1.f,.65f,.16f,.07f);fillCircle(x,y,r*1.85f,28);
    for(int i=0;i<(intercept?3:5);i++){
        float r2=r*(1.f-i*.18f),al=.86f-i*.15f;
        if(intercept)glColor4f(.16f,.55f,1.f,al);
        else glColor4f(1.f,.20f+i*.12f,0.f,al);
        fillCircle(x,y,r2,20);}
    glColor4f(1,1,.93f,1);fillCircle(x,y,r*.22f,12);
    glLineWidth(1.5f);
    int rays=intercept?8:12;
    glColor4f(intercept?.26f:1.f,intercept?.65f:.40f,intercept?1.f:.04f,.42f);
    glBegin(GL_LINES);
    for(int i=0;i<rays;i++){float a=2*PI*i/rays;
        glVertex2f(x,y);glVertex2f(x+cosf(a)*r*1.32f,y+sinf(a)*r*1.32f);}
    glEnd();glLineWidth(1.f);
}

void drawAllMissiles(){
    for(int i=0;i<MAX_MISSILES;i++){
        Missile*m=&missiles[i];
        if(m->expActive)drawExplosion(m->expX,m->expY,m->expR,m->state==MS_INTERCEPTED);
        if(m->state!=MS_FLYING)continue;
        drawMissileTrail(m);
        float pt=m->t-m->speed;if(pt<0)pt=0;
        float px,py;
        bezierPt(pt,m->startX,m->startY,m->ctrlX,m->ctrlY,m->endX,m->endY,&px,&py);
        float ddx=m->x-px,ddy=m->y-py;
        if(m->mtype==MT_AD){
            float c=(m->fromSide==SIDE_LEFT)?.26f:.68f;
            drawMissileShape(m->x,m->y,ddx,ddy,c,.46f,1.f,1);
        } else if(m->mtype==MT_AIR){
            // air-to-air missile: bright green
            drawMissileShape(m->x,m->y,ddx,ddy,.15f,.90f,.35f,2);
        } else {
            if(m->fromSide==SIDE_LEFT)
                drawMissileShape(m->x,m->y,ddx,ddy,.76f,.66f,.16f,0);
            else
                drawMissileShape(m->x,m->y,ddx,ddy,.72f,.16f,.16f,0);
        }
    }
}

void initAircraft(){
    aircraft[0].x=80;aircraft[0].y=510;aircraft[0].vx=1.3f;aircraft[0].vy=0;
    aircraft[0].active=1;aircraft[0].isLeft=1;aircraft[0].type=0;
    aircraft[0].rotorAngle=0;aircraft[0].bombTimer=0;aircraft[0].bombCooldown=randF(8,13);
    aircraft[0].hitTimer=0;aircraft[0].crashed=0;aircraft[0].crashX=0;aircraft[0].crashY=0;
    aircraft[0].airMissileTimer=0;aircraft[0].airMissileCooldown=randF(12,20);aircraft[0].fleeing=0;

    aircraft[1].x=W-80;aircraft[1].y=530;aircraft[1].vx=-1.2f;aircraft[1].vy=0;
    aircraft[1].active=1;aircraft[1].isLeft=0;aircraft[1].type=0;
    aircraft[1].rotorAngle=0;aircraft[1].bombTimer=0;aircraft[1].bombCooldown=randF(8,13);
    aircraft[1].hitTimer=0;aircraft[1].crashed=0;aircraft[1].crashX=0;aircraft[1].crashY=0;
    aircraft[1].airMissileTimer=0;aircraft[1].airMissileCooldown=randF(12,20);aircraft[1].fleeing=0;

    aircraft[2].x=160;aircraft[2].y=485;aircraft[2].vx=.9f;aircraft[2].vy=0;
    aircraft[2].active=1;aircraft[2].isLeft=1;aircraft[2].type=1;
    aircraft[2].rotorAngle=0;aircraft[2].bombTimer=0;aircraft[2].bombCooldown=randF(11,17);
    aircraft[2].hitTimer=0;aircraft[2].crashed=0;aircraft[2].crashX=0;aircraft[2].crashY=0;
    aircraft[2].airMissileTimer=0;aircraft[2].airMissileCooldown=randF(15,25);aircraft[2].fleeing=0;

    aircraft[3].x=W-160;aircraft[3].y=505;aircraft[3].vx=-.85f;aircraft[3].vy=0;
    aircraft[3].active=1;aircraft[3].isLeft=0;aircraft[3].type=1;
    aircraft[3].rotorAngle=0;aircraft[3].bombTimer=0;aircraft[3].bombCooldown=randF(11,17);
    aircraft[3].hitTimer=0;aircraft[3].crashed=0;aircraft[3].crashX=0;aircraft[3].crashY=0;
    aircraft[3].airMissileTimer=0;aircraft[3].airMissileCooldown=randF(15,25);aircraft[3].fleeing=0;
}

void dropBomb(Aircraft*ac){
    if(allBuildingsDestroyed) return;
    int idx=freeMissile();if(idx<0)return;
    int t=(ac->isLeft)?(3+randI(0,2)):randI(0,2);
    if(buildings[t].collapsed)return;
    float tx=buildings[t].x+buildings[t].w/2;
    float ty=buildings[t].y+buildings[t].h*.6f;
    float cx2=(ac->x+tx)/2+randF(-35,35),cy2=(ac->y+ty)/2;
    Side s=ac->isLeft?SIDE_LEFT:SIDE_RIGHT;
    initMissileS(&missiles[idx],ac->x,ac->y,tx,ty,cx2,cy2,randF(.0014f,.0022f),s,MT_ATTACK);
    spawnSmoke(ac->x,ac->y,2);
}

void drawJet(float x,float y,int goRight,float dmgAlpha){
    float d=goRight?1.f:-1.f;
    if(dmgAlpha>0){glColor4f(.05f,.04f,.03f,dmgAlpha);fillCircle(x,y,22,12);}
    glColor3f(.34f,.36f,.38f);
    glBegin(GL_QUADS);
    glVertex2f(x-26*d,y-4);glVertex2f(x+26*d,y-4);
    glVertex2f(x+26*d,y+4);glVertex2f(x-26*d,y+4);
    glEnd();
    glColor3f(.26f,.28f,.30f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x+26*d,y);glVertex2f(x+15*d,y-4);glVertex2f(x+15*d,y+4);
    glEnd();
    glColor4f(.26f,.62f,.82f,.62f);fillEllipse(x+8*d,y+4,6.5f,3.5f,12);
    glColor3f(.28f,.30f,.32f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x+3*d,y);glVertex2f(x-10*d,y+15);glVertex2f(x-17*d,y+15);
    glVertex2f(x+3*d,y);glVertex2f(x-10*d,y-15);glVertex2f(x-17*d,y-15);
    glEnd();
    glColor3f(.24f,.26f,.28f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x-18*d,y);glVertex2f(x-26*d,y+8);glVertex2f(x-26*d,y);
    glVertex2f(x-18*d,y);glVertex2f(x-26*d,y-8);glVertex2f(x-26*d,y);
    glEnd();
    glColor4f(1.f,.50f,.07f,.70f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x-26*d,y-3);glVertex2f(x-26*d,y+3);glVertex2f(x-38*d,y);
    glEnd();
    glColor4f(1.f,.80f,.40f,.50f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x-26*d,y-1.5f);glVertex2f(x-26*d,y+1.5f);glVertex2f(x-32*d,y);
    glEnd();
    glColor4f(.50f,.48f,.46f,.08f);
    for(int t2=1;t2<=3;t2++)fillCircle(x-26*d-t2*6*d,y,3.f+t2*1.2f,7);
}

void drawHelicopter(float x,float y,float ra,int goRight,float dmgAlpha){
    float d=goRight?1.f:-1.f;
    if(dmgAlpha>0){glColor4f(.06f,.05f,.03f,dmgAlpha);fillCircle(x,y,24,12);}
    glColor3f(.24f,.28f,.16f);fillEllipse(x,y,24*d,10,18);
    glColor4f(.20f,.60f,.80f,.60f);fillEllipse(x+10*d,y+2,10*d,7.5f,14);
    glColor3f(.20f,.24f,.13f);
    glBegin(GL_QUADS);
    glVertex2f(x-6*d,y-2);glVertex2f(x-38*d,y-1);
    glVertex2f(x-38*d,y+3);glVertex2f(x-6*d,y+3);
    glEnd();
    glColor3f(.18f,.21f,.12f);thickLine(x-38*d,y-4,x-38*d,y+7,2.f);
    float rx1=x+24*d*cosf(ra),ry1=y+10+24*sinf(ra);
    float rx2=x+24*d*cosf(ra+PI),ry2=y+10+24*sinf(ra+PI);
    glColor3f(.18f,.21f,.12f);thickLine(rx1,ry1,rx2,ry2,2.5f);
    float rx3=x+24*d*cosf(ra+PI*.5f),ry3=y+10+24*sinf(ra+PI*.5f);
    float rx4=x+24*d*cosf(ra+PI*1.5f),ry4=y+10+24*sinf(ra+PI*1.5f);
    thickLine(rx3,ry3,rx4,ry4,2.5f);
    glColor3f(.16f,.18f,.10f);
    thickLine(x-11*d,y-10,x+11*d,y-10,2.f);
    thickLine(x-8*d,y-4,x-8*d,y-10,1.8f);thickLine(x+8*d,y-4,x+8*d,y-10,1.8f);
    glColor4f(.85f,.85f,.65f,.09f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x+8*d,y-10);glVertex2f(x+26*d,y-52);glVertex2f(x-7*d,y-52);
    glEnd();
}

void drawCrashedAircraft(float x,float y,int type){
    if(type==0){
        glColor3f(.18f,.15f,.10f);fillEllipse(x,y,28,8,12);
        glColor3f(.25f,.20f,.14f);
        glBegin(GL_QUADS);
        glVertex2f(x-20,y);glVertex2f(x+20,y);
        glVertex2f(x+18,y+8);glVertex2f(x-18,y+8);
        glEnd();
    } else {
        glColor3f(.16f,.14f,.10f);fillEllipse(x,y,22,7,12);
        glColor3f(.20f,.16f,.10f);
        glBegin(GL_QUADS);
        glVertex2f(x-18,y);glVertex2f(x+18,y);
        glVertex2f(x+16,y+6);glVertex2f(x-16,y+6);
        glEnd();
        glColor3f(.28f,.24f,.16f);
        thickLine(x-15,y+2,x+15,y+2,3);
        thickLine(x,y-10,x,y+14,3);
    }
    float ff=.4f+.6f*sinf(globalTime*7.f);
    glColor4f(1.f,.30f,.0f,ff*.65f);fillCircle(x,y+12,9*ff,10);
    glColor4f(1.f,.65f,.0f,ff*.50f);fillCircle(x+4,y+17,5*ff,8);
    spawnSmoke(x+randF(-5,5),y+10,1);
}

// -----------------------------------------------------------------------
// updateAircraft: fix fleeing on war end, add air-to-air combat
// -----------------------------------------------------------------------
void updateAircraft(){
    // War ended: all aircraft flee immediately off screen
    if(allBuildingsDestroyed){
        for(int i=0;i<MAX_AIRCRAFT;i++){
            Aircraft*ac=&aircraft[i];
            if(!ac->active||ac->crashed) continue;
            if(!ac->fleeing){
                ac->fleeing=1;
                // Fly away from scene center (screen edge)
                ac->vx=(ac->isLeft)?-4.5f:4.5f;
                ac->vy=0;
                spawnSmoke(ac->x,ac->y,3);
            }
            ac->x+=ac->vx*gameSpeed;
            ac->rotorAngle+=.17f*gameSpeed;
            // Once off screen deactivate
            if(ac->x<-120||ac->x>W+120) ac->active=0;
        }
        return;
    }

    for(int i=0;i<MAX_AIRCRAFT;i++){
        Aircraft*ac=&aircraft[i];
        if(!ac->active)continue;
        if(ac->crashed)continue;

        if(ac->hitTimer>0){
            ac->hitTimer-=.016f*gameSpeed;
            ac->vy-=.28f*gameSpeed;
            ac->y+=ac->vy*gameSpeed;
            ac->x+=ac->vx*gameSpeed;
            ac->rotorAngle+=.06f*gameSpeed;
            spawnSmoke(ac->x,ac->y,2);
            if(ac->y<=GY){
                ac->crashed=1;
                ac->crashX=ac->x;
                ac->crashY=GY+5;
                spawnDebris(ac->x,GY,40,.22f,.18f,.10f);
                spawnSmoke(ac->x,GY,20);
                spawnDebris(ac->x,GY,25,1.f,.4f,.05f);
            }
            continue;
        }

        ac->x+=ac->vx*gameSpeed;
        ac->rotorAngle+=.17f*gameSpeed;
        if(ac->vx>0&&ac->x>W+80)ac->x=-80;
        if(ac->vx<0&&ac->x<-80)ac->x=W+80;

        // Bomb ground targets
        ac->bombTimer+=.016f*gameSpeed;
        if(ac->bombTimer>=ac->bombCooldown){
            dropBomb(ac);ac->bombTimer=0;
            ac->bombCooldown=randF(8,15);
        }

        // Air-to-air missile: find nearest enemy aircraft
        ac->airMissileTimer+=.016f*gameSpeed;
        if(ac->airMissileTimer>=ac->airMissileCooldown){
            ac->airMissileTimer=0;
            ac->airMissileCooldown=randF(10,18);
            // find nearest enemy aircraft
            float bestDist=9999;int bestTarget=-1;
            for(int j=0;j<MAX_AIRCRAFT;j++){
                if(j==i) continue;
                if(!aircraft[j].active||aircraft[j].crashed) continue;
                if(aircraft[j].isLeft==ac->isLeft) continue; // same side
                float dx=aircraft[j].x-ac->x;
                float dy=aircraft[j].y-ac->y;
                float dist=sqrtf(dx*dx+dy*dy);
                if(dist<bestDist){bestDist=dist;bestTarget=j;}
            }
            if(bestTarget>=0 && bestDist<600.f){
                launchAirToAirMissile(i, bestTarget);
            }
        }
    }
}

void drawAllAircraft(){
    for(int i=0;i<MAX_AIRCRAFT;i++){
        Aircraft*ac=&aircraft[i];
        if(!ac->active)continue;
        if(ac->crashed){
            drawCrashedAircraft(ac->crashX,ac->crashY,ac->type);
            continue;
        }
        float dmgA=(ac->hitTimer>0)?(1.f-ac->hitTimer)*0.5f:0.f;
        int goRight=(ac->vx>0);
        if(ac->type==0) drawJet(ac->x,ac->y,goRight,dmgA);
        else            drawHelicopter(ac->x,ac->y,ac->rotorAngle,goRight,dmgA);
    }
}

void initCivilians(){
    for(int i=0;i<4;i++){
        civs[i].x=randF(60,340);civs[i].y=GY;
        civs[i].vx=randF(.8f,1.4f);civs[i].legPhase=randF(0,2*PI);
        civs[i].active=1;civs[i].dir=(i%2==0)?1:-1;
        civs[i].skin=randI(0,2);civs[i].isLeft=1;civs[i].fleeing=0;}
    for(int i=4;i<MAX_CIVILIANS;i++){
        civs[i].x=randF(860,1140);civs[i].y=GY;
        civs[i].vx=randF(.8f,1.4f);civs[i].legPhase=randF(0,2*PI);
        civs[i].active=1;civs[i].dir=(i%2==0)?1:-1;
        civs[i].skin=randI(0,2);civs[i].isLeft=0;civs[i].fleeing=0;}
}

void drawCivilian(Civilian*c){
    if(!c->active)return;
    float x=c->x,fy=c->y;
    float sR[]={.88f,.72f,.50f},sG[]={.68f,.52f,.34f},sB[]={.58f,.42f,.28f};
    int sk=c->skin;
    float ls=sinf(c->legPhase)*8.f;
    float as2=sinf(c->legPhase)*5.f;
    glColor4f(0,0,0,.15f);fillEllipse(x,fy,6,2,10);
    glColor3f(.18f,.18f,.28f);
    thickLine(x-3,fy,x-3+ls*.3f,fy+13,3.f);
    thickLine(x+3,fy,x+3-ls*.3f,fy+13,3.f);
    float cR=c->isLeft?.36f:.18f;
    glColor3f(cR,.32f,.48f);
    glBegin(GL_QUADS);
    glVertex2f(x-5,fy+13);glVertex2f(x+5,fy+13);
    glVertex2f(x+4,fy+26);glVertex2f(x-4,fy+26);
    glEnd();
    glColor3f(sR[sk],sG[sk],sB[sk]);
    thickLine(x-5,fy+24,x-10-as2*.35f,fy+17,2.4f);
    thickLine(x+5,fy+24,x+10+as2*.35f,fy+17,2.4f);
    thickLine(x,fy+26,x,fy+30,2.8f);
    fillCircle(x,fy+35,5.f,14);
    glColor3f(.13f,.08f,.04f);fillEllipse(x,fy+38,4.8f,2.8f,10);
}

void updateCivilians(){
    int danger=0;
    for(int m=0;m<MAX_MISSILES;m++)
        if(missiles[m].state==MS_FLYING&&missiles[m].mtype==MT_ATTACK){danger=1;break;}
    for(int i=0;i<MAX_CIVILIANS;i++){
        if(!civs[i].active)continue;
        Civilian*c=&civs[i];
        if(danger&&!c->fleeing){c->fleeing=1;c->dir=(c->x<W/2)?-1:1;c->vx=randF(2.4f,3.6f);}
        c->x+=c->dir*c->vx*gameSpeed;
        c->legPhase+=c->vx*.22f*gameSpeed;
        if(c->fleeing){if(c->x<-65||c->x>W+65)c->active=0;}
        else{
            if(c->isLeft){if(c->x<38)c->dir=1;if(c->x>348)c->dir=-1;}
            else{if(c->x<858)c->dir=1;if(c->x>1152)c->dir=-1;}}
    }
}

void drawBarricade(float x,float y){
    glColor3f(.44f,.42f,.39f);
    glBegin(GL_QUADS);
    glVertex2f(x,y);glVertex2f(x+24,y);
    glVertex2f(x+22,y+17);glVertex2f(x+2,y+17);
    glEnd();
    glColor3f(.53f,.51f,.48f);
    glBegin(GL_QUADS);
    glVertex2f(x+2,y+17);glVertex2f(x+22,y+17);
    glVertex2f(x+20,y+27);glVertex2f(x+4,y+27);
    glEnd();
    glColor4f(1,1,1,.06f);
    glBegin(GL_QUADS);
    glVertex2f(x+4,y+17);glVertex2f(x+10,y+17);
    glVertex2f(x+9,y+27);glVertex2f(x+4,y+27);
    glEnd();
}

void drawSandbagWall(float bx,float by,int n){
    for(int i=0;i<n;i++){
        glColor3f(.42f,.34f,.17f);fillEllipse(bx+i*18,by,9,5.5f,12);
        glColor3f(.52f,.44f,.23f);fillEllipse(bx+i*18-1,by+2,3.8f,2.5f,8);}
    for(int i=0;i<n-1;i++){
        glColor3f(.38f,.30f,.15f);fillEllipse(bx+9+i*18,by+9.5f,9,5.5f,12);}
}

void drawBarbedWire(float x1,float y1,float x2,float y2){
    glColor3f(.40f,.36f,.26f);lineDDA(x1,y1,x2,y2);
    float dx=x2-x1,dy=y2-y1,len=sqrtf(dx*dx+dy*dy);
    int sp=(int)(len/11);
    for(int i=0;i<sp;i++){
        float t2=(float)i/sp,bx=x1+t2*dx,by=y1+t2*dy;
        glColor3f(.50f,.46f,.36f);
        lineDDA(bx-3,by-3,bx+3,by+3);lineDDA(bx+3,by-3,bx-3,by+3);}
}

void drawSky(){
    glBegin(GL_QUADS);
    glColor3f(.010f,.013f,.068f);glVertex2f(0,H);glVertex2f(W,H);
    glColor3f(.030f,.040f,.140f);glVertex2f(W,H*.50f);glVertex2f(0,H*.50f);
    glEnd();
    glBegin(GL_QUADS);
    glColor3f(.030f,.040f,.140f);glVertex2f(0,H*.50f);glVertex2f(W,H*.50f);
    glColor3f(.044f,.055f,.150f);glVertex2f(W,H*.28f);glVertex2f(0,H*.28f);
    glEnd();
    glBegin(GL_QUADS);
    glColor3f(.044f,.055f,.150f);glVertex2f(0,H*.28f);glVertex2f(W,H*.28f);
    glColor3f(.050f,.062f,.155f);glVertex2f(W,0);glVertex2f(0,0);
    glEnd();
}

void drawStars(){
    for(int i=0;i<NUM_STARS;i++){
        float tw=.34f+.66f*sinf(globalTime*1.3f+starPhase[i]);
        float sz=starSz[i];
        if(sz>2.0f){
            float s=sz*1.4f;
            glColor4f(.92f,.90f,.80f,tw*.74f);
            lineDDA(starX[i]-s,starY[i],starX[i]+s,starY[i]);
            lineDDA(starX[i],starY[i]-s,starX[i],starY[i]+s);
            glColor4f(1,1,.93f,tw*.84f);fillCircle(starX[i],starY[i],sz*.27f,6);
        } else if(sz>1.3f){
            float s=sz*1.1f;
            glColor4f(.90f,.90f,.84f,tw*.58f);
            lineDDA(starX[i]-s,starY[i],starX[i]+s,starY[i]);
            lineDDA(starX[i],starY[i]-s,starX[i],starY[i]+s);
            glColor4f(1,1,1,tw*.66f);fillCircle(starX[i],starY[i],sz*.20f,5);
        } else {
            glColor4f(.80f,.80f,.86f,tw*.48f);fillCircle(starX[i],starY[i],sz*.34f,5);}
    }
}

void drawMoon(){
    float mx=W*.70f,my=H-88.f,mr=26.f;
    glColor4f(.72f,.70f,.50f,.026f);fillCircle(mx,my,mr+32,26);
    glColor3f(.64f,.62f,.45f);fillCircle(mx,my,mr,46);
    for(int r2=(int)mr;r2>(int)(mr*.58f);r2--){
        float f=(mr-(float)r2)/(mr*.42f);
        glColor4f(.09f,.07f,.03f,f*.14f);circleMid(mx,my,(float)r2);}
    glColor4f(.20f,.17f,.06f,.68f);circleMid(mx,my,mr);circleMid(mx,my,mr+1.f);
    float crd[][4]={{mx-10,my+8,6,4},{mx+12,my-9,5,3.5f},
                    {mx+2,my+15,4.5f,3.f},{mx-17,my-5,3.5f,2.5f}};
    for(int i=0;i<4;i++){
        glColor4f(.52f,.48f,.34f,1);fillCircle(crd[i][0],crd[i][1],crd[i][2],12);
        glColor4f(.62f,.58f,.42f,.56f);
        fillCircle(crd[i][0]-crd[i][2]*.18f,crd[i][1]+crd[i][2]*.18f,crd[i][3],10);}
}

void initClouds2(){
    float xs[]={-200,380,660,940,180,1080,-120,520,300,810};
    float ys[]={H-148,H-128,H-143,H-133,H-163,H-123,H-153,H-168,H-138,H-158};
    float sp[]={.11f,-.09f,.13f,-.10f,.08f,-.07f,.12f,-.08f,.09f,-.11f};
    float sc[]={1.1f,.85f,.92f,1.0f,.75f,.88f,1.05f,.78f,.95f,.82f};
    float al[]={.58f,.54f,.60f,.56f,.52f,.62f,.58f,.55f,.60f,.57f};
    for(int i=0;i<NUM_CLOUDS;i++){
        clouds[i].x=xs[i];clouds[i].y=ys[i];
        clouds[i].spd=sp[i];clouds[i].sc=sc[i];clouds[i].alpha=al[i];}
}
void updateClouds(){
    for(int i=0;i<NUM_CLOUDS;i++){
        clouds[i].x+=clouds[i].spd*gameSpeed;
        if(clouds[i].spd>0&&clouds[i].x>W+230)clouds[i].x=-240;
        if(clouds[i].spd<0&&clouds[i].x<-240)clouds[i].x=W+230;}
}
void drawOneCloud(float x,float y,float sc,float al){
    glColor4f(.09f,.10f,.20f,al);
    fillCircle(x,y,25*sc,16);fillCircle(x+29*sc,y+6*sc,20*sc,14);
    fillCircle(x-27*sc,y+4*sc,18*sc,14);fillCircle(x+11*sc,y+15*sc,15*sc,12);
    fillCircle(x-9*sc,y+16*sc,13*sc,12);fillCircle(x+42*sc,y+1*sc,13*sc,12);
    fillCircle(x-42*sc,y+3*sc,12*sc,12);
    glColor4f(.26f,.28f,.42f,al*.16f);
    fillCircle(x,y+5*sc,15*sc,12);fillCircle(x+27*sc,y+9*sc,10*sc,10);}
void drawClouds2(){
    for(int i=0;i<NUM_CLOUDS;i++)
        drawOneCloud(clouds[i].x,clouds[i].y,clouds[i].sc,clouds[i].alpha);}

void drawGround(){
    glBegin(GL_QUADS);
    glColor3f(.055f,.105f,.036f);glVertex2f(0,GY);glVertex2f(490,GY);
    glColor3f(.080f,.150f,.052f);glVertex2f(490,0);glVertex2f(0,0);
    glEnd();
    glBegin(GL_QUADS);
    glColor3f(.048f,.092f,.030f);glVertex2f(710,GY);glVertex2f(W,GY);
    glColor3f(.070f,.132f,.046f);glVertex2f(W,0);glVertex2f(710,0);
    glEnd();
    glColor4f(.046f,.036f,.020f,.36f);
    fillEllipse(75,90,40,13,14);fillEllipse(305,72,36,11,14);
    fillEllipse(190,42,28,9,12);
    fillEllipse(915,88,38,12,14);fillEllipse(1075,68,33,10,14);
    fillEllipse(1015,40,26,8,12);
    glColor4f(.028f,.022f,.013f,.52f);
    fillEllipse(375,125,22,7,14);fillEllipse(175,105,17,6,12);
    fillEllipse(845,120,20,7,14);fillEllipse(1035,104,16,5,12);
    glColor4f(.082f,.065f,.038f,.28f);
    circleMid(375,125,22);circleMid(845,120,20);
    glColor4f(.036f,.028f,.016f,.28f);
    lineDDA(52,GY,125,78);lineDDA(62,GY,135,78);
    lineDDA(1065,GY,990,78);lineDDA(1075,GY,1000,78);
    float rks[]={58,175,285,345,425,835,895,955,1075,1145};
    float rky[]={152,142,157,135,165,148,145,162,138,155};
    for(int i=0;i<10;i++){
        glColor3f(.26f+i%3*.03f,.24f,.19f);
        fillEllipse(rks[i],rky[i],3+i%3*2,2.5f+i%2,8);}
    glColor4f(.13f,.19f,.06f,.55f);
    float gx[]={88,172,256,336,410,865,940,1016,1096,1165};
    float gy2[]={178,170,174,168,172,170,178,165,172,168};
    for(int i=0;i<10;i++){
        lineDDA(gx[i],gy2[i],gx[i]-4,gy2[i]+11);
        lineDDA(gx[i],gy2[i],gx[i]+3,gy2[i]+13);
        lineDDA(gx[i],gy2[i],gx[i]+7,gy2[i]+9);}
}

void drawRiver(){
    glColor3f(.035f,.125f,.285f);
    glBegin(GL_QUADS);
    glVertex2f(490,GY);glVertex2f(710,GY);
    glVertex2f(860,0);glVertex2f(340,0);
    glEnd();

    for(int i=0;i<12;i++){
        float t0=(float)i/12,t1=(float)(i+1)/12;
        float lx0=490+(340-490)*t0,rx0=710+(860-710)*t0;
        float lx1=490+(340-490)*t1,rx1=710+(860-710)*t1;
        float y0=(1-t0)*GY,y1=(1-t1)*GY;
        glColor4f(.038f+t0*.042f,.128f+t0*.085f,.288f+t0*.095f,.45f);
        glBegin(GL_QUADS);
        glVertex2f(lx0,y0);glVertex2f(rx0,y0);
        glVertex2f(rx1,y1);glVertex2f(lx1,y1);
        glEnd();
    }

    glColor4f(.048f,.165f,.345f,.65f);
    glBegin(GL_QUADS);
    glVertex2f(340,0);glVertex2f(860,0);
    glVertex2f(830,22);glVertex2f(370,22);
    glEnd();

    int waveOffset=(int)(globalTime*25)%30;
    glColor4f(.145f,.245f,.425f,.28f);
    for(int i=0;i<15;i++){
        float t2=(float)i/14;
        float lx=490+(340-490)*t2;
        float rx=710+(860-710)*t2;
        float wy=(1-t2)*GY*.96f;
        float wavePhase=sinf(globalTime*2.f+i*.5f);
        glBegin(GL_LINE_STRIP);
        for(int j=0;j<8;j++){
            float px=lx+(rx-lx)*j/7.f;
            float py=wy+wavePhase*2.5f*sinf(j*.8f+globalTime*3.f);
            glVertex2f(px,py);
        }
        glEnd();
    }

    glColor4f(.65f,.72f,.82f,.18f);
    for(int i=0;i<10;i++){
        float t2=(float)i/9;
        float lx=490+(340-490)*t2+waveOffset;
        float rx=710+(860-710)*t2-waveOffset;
        float wy=(1-t2)*GY*.94f;
        float foam=2+sinf(globalTime*4.f+i)*1.5f;
        fillEllipse(lx+15,wy+foam,8+t2*4,2+t2,8);
        fillEllipse(rx-15,wy-foam,8+t2*4,2+t2,8);
    }

    float moonReflectX=W*.70f;
    float moonReflectY=GY*.45f;
    float moonShimmer=.15f+.12f*sinf(globalTime*1.3f);
    glColor4f(.78f,.75f,.52f,moonShimmer*.35f);
    fillEllipse(moonReflectX-10,moonReflectY,22,65,20);
    glColor4f(.88f,.85f,.62f,moonShimmer*.25f);
    fillEllipse(moonReflectX-5,moonReflectY,15,55,16);

    glBegin(GL_QUADS);
    glColor4f(.048f,.165f,.315f,.55f);
    glVertex2f(490,GY);glVertex2f(500,GY);
    glColor4f(.075f,.155f,.062f,.0f);
    glVertex2f(385,0);glVertex2f(340,0);
    glEnd();
    glBegin(GL_QUADS);
    glColor4f(.048f,.165f,.315f,.55f);
    glVertex2f(710,GY);glVertex2f(700,GY);
    glColor4f(.068f,.142f,.055f,.0f);
    glVertex2f(815,0);glVertex2f(860,0);
    glEnd();

    float leftRocks[]={495,502,512,525,538,552,468,478,485};
    float leftRockY[]={GY-2,GY-1,GY-3,GY-2,GY-1,GY-2,GY-1,GY-3,GY-2};
    for(int i=0;i<9;i++){
        glColor3f(.32f+i%3*.04f,.30f,.24f);
        fillEllipse(leftRocks[i],leftRockY[i],4+i%3*2,3+i%2,10);
    }

    float rightRocks[]={705,698,688,675,662,648,718,728,738};
    float rightRockY[]={GY-2,GY-1,GY-3,GY-2,GY-1,GY-2,GY-1,GY-3,GY-2};
    for(int i=0;i<9;i++){
        glColor3f(.30f+i%3*.04f,.28f,.22f);
        fillEllipse(rightRocks[i],rightRockY[i],4+i%3*2,3+i%2,10);
    }

    glColor4f(.16f,.24f,.08f,.72f);
    float reedPos[]={493,505,518,535,548,687,698,710,723,735};
    for(int i=0;i<10;i++){
        float rx=reedPos[i];
        float sway=sinf(globalTime*1.5f+i*.7f)*2.f;
        lineDDA(rx,GY-3,rx+sway,GY+12);
        lineDDA(rx+3,GY-2,rx+3+sway,GY+14);
        glColor4f(.28f,.16f,.08f,.85f);
        fillEllipse(rx+sway,GY+13,1.5f,5,8);
        fillEllipse(rx+3+sway,GY+15,1.5f,5,8);
        glColor4f(.16f,.24f,.08f,.72f);
    }
}

void drawFighterShip(FighterShip* s){
    if(!s->active) return;

    float x = s->x;
    float y = s->y;

    float alpha = 1.0f;
    if(s->phase == 2){
        y -= s->sinkDepth;
        float fadeStart = GY * 0.35f;
        if(y < fadeStart){
            alpha = clampF(y / fadeStart, 0.f, 1.f);
        }
        if(alpha <= 0.f) return;
    }

    if(s->destroyed){
        float sinkD = s->destroyTimer * 15;
        y -= sinkD;
        float fireFlicker = .5f + .5f * sinf(globalTime * 9.f);
        glColor4f(1.f, .4f, .05f, fireFlicker * .7f);
        fillCircle(x, y + 18, 12 * fireFlicker, 12);
        glColor4f(1.f, .75f, .1f, fireFlicker * .5f);
        fillCircle(x + 5, y + 22, 7 * fireFlicker, 10);
        spawnSmoke(x + randF(-8, 8), y + 15, 2);
        return;
    }

    int dir = 1;

    glColor4f(0, 0, 0, .25f * alpha);
    fillEllipse(x, y - 3, 38, 8, 14);

    glColor4f(.18f, .20f, .22f, alpha);
    glBegin(GL_POLYGON);
    glVertex2f(x - 35 * dir, y);
    glVertex2f(x + 35 * dir, y);
    glVertex2f(x + 32 * dir, y + 12);
    glVertex2f(x - 28 * dir, y + 12);
    glEnd();

    glColor4f(.22f, .24f, .26f, alpha);
    glBegin(GL_QUADS);
    glVertex2f(x - 28 * dir, y + 12);
    glVertex2f(x + 32 * dir, y + 12);
    glVertex2f(x + 28 * dir, y + 20);
    glVertex2f(x - 24 * dir, y + 20);
    glEnd();

    glColor4f(.25f, .27f, .29f, alpha);
    glBegin(GL_QUADS);
    glVertex2f(x - 8 * dir, y + 20);
    glVertex2f(x + 12 * dir, y + 20);
    glVertex2f(x + 10 * dir, y + 32);
    glVertex2f(x - 6 * dir, y + 32);
    glEnd();

    glColor4f(.35f, .75f, .95f, .75f * alpha);
    fillCircle(x + 2 * dir, y + 26, 2.5f, 8);
    fillCircle(x + 7 * dir, y + 25, 2.2f, 8);

    glColor4f(.28f, .30f, .32f, alpha);
    fillCircle(x + 22 * dir, y + 16, 5, 12);
    glColor4f(.32f, .34f, .36f, alpha);
    thickLine(x + 22 * dir, y + 16, x + 38 * dir, y + 18, 3.5f);

    glColor4f(.26f, .28f, .30f, alpha);
    glBegin(GL_QUADS);
    glVertex2f(x - 18 * dir, y + 20);
    glVertex2f(x - 10 * dir, y + 20);
    glVertex2f(x - 10 * dir, y + 28);
    glVertex2f(x - 18 * dir, y + 28);
    glEnd();

    glColor4f(.40f, .42f, .44f, alpha);
    fillCircle(x - 2 * dir, y + 33, 3, 10);
    thickLine(x - 2 * dir, y + 32, x - 2 * dir, y + 38, 1.5f);

    float flagWave = sinf(globalTime * 3.f + x * .01f);
    glColor4f(.72f, .16f, .16f, alpha);
    glBegin(GL_TRIANGLES);
    glVertex2f(x + 10 * dir, y + 32);
    glVertex2f(x + 10 * dir + 8 * dir, y + 35 + flagWave);
    glVertex2f(x + 10 * dir, y + 38);
    glEnd();

    glColor4f(.35f, .45f, .55f, .25f * alpha);
    for(int i = 0; i < 3; i++){
        float wx = x - (15 + i * 8) * dir;
        float wy = y - 2 + sinf(globalTime * 2.f + i) * 2;
        fillEllipse(wx, wy, 12 - i * 2, 4, 10);
    }

    if(s->health < 1.f){
        glColor4f(.2f, .2f, .2f, .6f * alpha);
        glBegin(GL_QUADS);
        glVertex2f(x - 20, y + 38);
        glVertex2f(x + 20, y + 38);
        glVertex2f(x + 20, y + 42);
        glVertex2f(x - 20, y + 42);
        glEnd();

        glColor4f(s->health > .5f ? .3f : 1.f,
                  s->health > .5f ? .8f : .3f,
                  .2f, alpha);
        glBegin(GL_QUADS);
        glVertex2f(x - 20, y + 38);
        glVertex2f(x - 20 + 40 * s->health, y + 38);
        glVertex2f(x - 20 + 40 * s->health, y + 42);
        glVertex2f(x - 20, y + 42);
        glEnd();
    }
}

// -----------------------------------------------------------------------
// drawWindmill: damaged state when nearby explosion shakes blades faster
// -----------------------------------------------------------------------
void drawWindmill(Windmill* wm){
    float x = wm->x;
    float y = wm->y;

    // Tower color darkens if damaged
    float dmgFactor = wm->damaged ? 0.65f : 1.0f;

    glColor3f(.82f*dmgFactor, .78f*dmgFactor, .70f*dmgFactor);
    glBegin(GL_QUADS);
    glVertex2f(x - 12, y);
    glVertex2f(x + 12, y);
    glVertex2f(x + 8, y + 65);
    glVertex2f(x - 8, y + 65);
    glEnd();

    if(wm->damaged){
        // Cracks on tower
        glColor4f(.15f,.10f,.05f,.65f);
        lineDDA(x-4, y+10, x+2, y+35);
        lineDDA(x+3, y+20, x-2, y+50);
    }

    glColor4f(.25f, .22f, .18f, .85f);
    fillCircle(x, y + 35, 4, 10);
    fillCircle(x, y + 50, 4, 10);

    glColor3f(.28f*dmgFactor, .24f*dmgFactor, .18f*dmgFactor);
    glBegin(GL_QUADS);
    glVertex2f(x - 5, y);
    glVertex2f(x + 5, y);
    glVertex2f(x + 5, y + 18);
    glVertex2f(x - 5, y + 18);
    glEnd();

    glColor3f(.65f, .28f, .22f);
    glBegin(GL_TRIANGLES);
    glVertex2f(x, y + 75);
    glVertex2f(x - 12, y + 65);
    glVertex2f(x + 12, y + 65);
    glEnd();

    glColor3f(.35f, .32f, .28f);
    fillCircle(x, y + 65, 6, 16);

    for(int i = 0; i < 4; i++){
        float angle = wm->bladeAngle + i * PI / 2;
        float bx = x + cosf(angle) * 28;
        float by = y + 65 + sinf(angle) * 28;

        // Blades appear slightly broken if damaged
        float bladeColor = wm->damaged ? 0.65f : 0.88f;
        glColor3f(bladeColor, bladeColor*0.97f, bladeColor*0.89f);
        glBegin(GL_TRIANGLES);
        glVertex2f(x, y + 65);
        glVertex2f(bx + cosf(angle + PI / 2) * 4, by + sinf(angle + PI / 2) * 4);
        glVertex2f(bx + cosf(angle - PI / 2) * 4, by + sinf(angle - PI / 2) * 4);
        glEnd();

        glColor4f(.45f, .42f, .38f, .7f);
        lineDDA(x, y + 65, bx, by);
    }

    // Smoke from damaged windmill
    if(wm->damaged && (int)(globalTime*30)%5==0){
        spawnSmoke(x, y+65, 1);
    }
}

void drawTree(Tree* tree){
    float x = tree->x;
    float y = tree->y;
    float h = tree->height;

    glColor3f(.35f, .25f, .15f);
    glBegin(GL_QUADS);
    glVertex2f(x - tree->width / 2, y);
    glVertex2f(x + tree->width / 2, y);
    glVertex2f(x + tree->width / 2.5f, y + h * .6f);
    glVertex2f(x - tree->width / 2.5f, y + h * .6f);
    glEnd();

    if(tree->leafType == 0){
        glColor4f(.18f, .32f, .12f, .85f);
        fillCircle(x, y + h * .55f, h * .35f, 20);
        fillCircle(x - h * .2f, y + h * .68f, h * .28f, 18);
        fillCircle(x + h * .2f, y + h * .68f, h * .28f, 18);
        fillCircle(x, y + h * .82f, h * .25f, 16);
        glColor4f(.28f, .45f, .20f, .55f);
        fillCircle(x - h * .1f, y + h * .75f, h * .15f, 12);
    } else {
        glColor4f(.15f, .28f, .10f, .88f);
        glBegin(GL_TRIANGLES);
        glVertex2f(x, y + h);
        glVertex2f(x - h * .4f, y + h * .5f);
        glVertex2f(x + h * .4f, y + h * .5f);
        glEnd();
        glColor4f(.17f, .30f, .11f, .85f);
        glBegin(GL_TRIANGLES);
        glVertex2f(x, y + h * .85f);
        glVertex2f(x - h * .35f, y + h * .45f);
        glVertex2f(x + h * .35f, y + h * .45f);
        glEnd();
        glColor4f(.19f, .32f, .13f, .82f);
        glBegin(GL_TRIANGLES);
        glVertex2f(x, y + h * .7f);
        glVertex2f(x - h * .3f, y + h * .4f);
        glVertex2f(x + h * .3f, y + h * .4f);
        glEnd();
    }
}

void initShip(){
    ship.x = 600;
    ship.y = -50;
    ship.vy = 0.8f;
    ship.health = 1.f;
    ship.active = 1;
    ship.destroyed = 0;
    ship.destroyTimer = 0;
    ship.missileTimer = 0;
    ship.missileCooldown = randF(2.5f, 4.f);
    ship.phase = 0;
    ship.retreatVx = 0.f;
    ship.retreatVy = 0.f;
    ship.sinkDepth = 0.f;
}

void initWindmills(){
    windmills[0].x = 420;
    windmills[0].y = GY;
    windmills[0].bladeAngle = 0;
    windmills[0].spinSpeed = 0.025f;
    windmills[0].damaged = 0;
    windmills[0].damageTimer = 0;
    windmills[0].isLeft = 1;

    windmills[1].x = 780;
    windmills[1].y = GY;
    windmills[1].bladeAngle = PI / 4;
    windmills[1].spinSpeed = 0.025f;
    windmills[1].damaged = 0;
    windmills[1].damageTimer = 0;
    windmills[1].isLeft = 0;
}

void initTrees(){
    trees[0] = (Tree){70, GY, 45, 8, 0};
    trees[1] = (Tree){155, GY, 52, 9, 1};
    trees[2] = (Tree){240, GY, 38, 7, 0};
    trees[3] = (Tree){320, GY, 48, 8, 1};
    trees[4] = (Tree){385, GY, 42, 7, 0};
    trees[5] = (Tree){450, GY, 35, 6, 1};

    trees[6] = (Tree){750, GY, 40, 7, 0};
    trees[7] = (Tree){815, GY, 48, 8, 1};
    trees[8] = (Tree){895, GY, 44, 7, 0};
    trees[9] = (Tree){975, GY, 51, 9, 1};
    trees[10] = (Tree){1050, GY, 37, 6, 0};
    trees[11] = (Tree){1130, GY, 46, 8, 1};
}

void updateShip(){
    if(!ship.active) return;

    if(ship.destroyed){
        ship.destroyTimer += .016f * gameSpeed;
        if(ship.destroyTimer > 3.f){
            ship.active = 0;
        }
        return;
    }

    if(ship.phase == 0){
        ship.y += ship.vy * gameSpeed;
        if(ship.y >= GY * 0.60f){
            ship.phase = 1;
            ship.vy = 0;
        }
    }
    else if(ship.phase == 1){
        ship.missileTimer += .016f * gameSpeed;
        if(ship.missileTimer >= ship.missileCooldown){
            launchShipMissile();
            ship.missileTimer = 0;
            ship.missileCooldown = randF(2.f, 3.5f);
        }

        if(allBuildingsDestroyed){
            ship.phase = 2;
            float riverTopX = 600.f;
            float dx = riverTopX - ship.x;
            float dist = fabsf(dx);
            ship.retreatVx = (dist > 1.f) ? (dx / (4.f / .016f)) : 0.f;
            ship.retreatVy = 0.f;
            ship.sinkDepth = 0.f;

            spawnSmoke(ship.x, ship.y + 20, 8);
            spawnDebris(ship.x, ship.y, 20, .4f, .6f, .9f);
        }
    }
    else if(ship.phase == 2){
        ship.x += ship.retreatVx * gameSpeed;
        ship.y -= 0.55f * gameSpeed;
        ship.sinkDepth += 0.30f * gameSpeed;

        if((int)(globalTime * 60) % 3 == 0){
            spawnSmoke(ship.x + randF(-12, 12), ship.y + 20, 2);
        }

        if(ship.y < -80.f || ship.sinkDepth > 60.f){
            ship.active = 0;
        }
    }
}

// -----------------------------------------------------------------------
// updateWindmills: react to nearby explosions
// -----------------------------------------------------------------------
void updateWindmills(){
    for(int i = 0; i < MAX_WINDMILLS; i++){
        Windmill* wm = &windmills[i];

        // Check if any explosion is close to this windmill
        for(int j=0; j<MAX_MISSILES; j++){
            if(!missiles[j].expActive) continue;
            float dx = missiles[j].expX - wm->x;
            float dy = missiles[j].expY - (wm->y + 65);  // hub height
            float dist = sqrtf(dx*dx + dy*dy);
            if(dist < 120.f){
                // Nearby explosion: speed up blades dramatically, mark damaged
                wm->spinSpeed = 0.18f + (120.f - dist) / 120.f * 0.22f;
                wm->damageTimer = 120.f;  // frames of fast spin
                wm->damaged = 1;
            }
        }

        // Gradually slow blades back down after explosion effect
        if(wm->damageTimer > 0){
            wm->damageTimer -= gameSpeed;
            // speed decays toward damaged idle speed
            if(wm->damageTimer <= 0){
                wm->spinSpeed = 0.055f;  // damaged = slightly faster than normal forever
            }
        } else if(!wm->damaged){
            wm->spinSpeed = 0.025f;  // normal speed
        }

        wm->bladeAngle += wm->spinSpeed * gameSpeed;
        if(wm->bladeAngle > 2 * PI)
            wm->bladeAngle -= 2 * PI;
    }
}

void drawText(float x,float y,const char*s,void*font){
    glRasterPos2f(x,y);for(const char*c=s;*c;c++)glutBitmapCharacter(font,*c);}

void drawHUD(){
    glColor4f(0,0,0,.65f);
    glBegin(GL_QUADS);
    glVertex2f(0,H-36);glVertex2f(W,H-36);glVertex2f(W,H);glVertex2f(0,H);
    glEnd();
    glColor4f(.55f,.55f,.60f,.15f);
    lineDDA(W/2-148,H-36,W/2-148,H);lineDDA(W/2+148,H-36,W/2+148,H);
    glColor3f(.85f,.67f,.13f);drawText(12,H-23,"< NATION ALPHA",GLUT_BITMAP_HELVETICA_18);
    char buf[64];glColor3f(.93f,.93f,.93f);
    sprintf(buf,"Buildings: %d/3",3-(leftScore>3?3:leftScore));drawText(12,H-10,buf,GLUT_BITMAP_8_BY_13);
    glColor3f(.24f,.57f,1.f);drawText(W-180,H-23,"NATION BETA >",GLUT_BITMAP_HELVETICA_18);
    sprintf(buf,"Buildings: %d/3",3-(rightScore>3?3:rightScore));glColor3f(.93f,.93f,.93f);drawText(W-180,H-10,buf,GLUT_BITMAP_8_BY_13);
    glColor3f(1.f,.18f,.12f);drawText(W/2-76,H-23,"WAR SIMULATION",GLUT_BITMAP_HELVETICA_18);

    if(allBuildingsDestroyed){
        glColor4f(0,0,0,.72f);
        glBegin(GL_QUADS);
        glVertex2f(W/2-170, H/2-20);glVertex2f(W/2+170, H/2-20);
        glVertex2f(W/2+170, H/2+40);glVertex2f(W/2-170, H/2+40);
        glEnd();
        glColor3f(1.f,.3f,.1f);
        drawText(W/2-145, H/2+22, "ALL TARGETS DESTROYED!", GLUT_BITMAP_HELVETICA_18);
        glColor3f(.9f,.85f,.2f);
        drawText(W/2-115, H/2+5,  "SHIP RETREATING INTO RIVER...", GLUT_BITMAP_HELVETICA_12);
    }

    glColor4f(0,0,0,.55f);
    glBegin(GL_QUADS);glVertex2f(0,0);glVertex2f(W,0);glVertex2f(W,22);glVertex2f(0,22);glEnd();
    glColor3f(.46f,.82f,.28f);
    drawText(10,6,"1:Slow  2:Normal  3:Fast  P:Pause  R:Reset  ESC:Quit",
             GLUT_BITMAP_8_BY_13);
    char sp[32];sprintf(sp,"x%.1f%s",gameSpeed,paused?" [PAUSED]":"");
    glColor3f(1.f,.75f,.11f);drawText(W-84,6,sp,GLUT_BITMAP_8_BY_13);
}

void initScene(){
    srand((unsigned)time(NULL));
    memset(missiles,0,sizeof(missiles));
    memset(debris,0,sizeof(debris));
    memset(smokeP,0,sizeof(smokeP));
    numPads=0;numTanks=0;
    allBuildingsDestroyed = 0;
    warJustEnded = 0;
    for(int i=0;i<NUM_STARS;i++){
        starX[i]=randF(10,W-10);starY[i]=randF(H*.36f,H-4);
        starPhase[i]=randF(0,2*PI);starSz[i]=randF(.6f,2.4f);}
    initBuildings();
    initCivilians();
    initClouds2();
    initAircraft();
    initShip();
    initWindmills();
    initTrees();
    leftMissileTimer=rightMissileTimer=0;
    leftMissileDelay=4.5f;rightMissileDelay=4.2f;
    tankTimerL=tankTimerR=0;
    tankDelayL=randF(8,13);tankDelayR=randF(9,14);
    leftScore = rightScore = 0;
}
void resetScene(){
    initScene();globalTime=0;paused=0;}

void display(){
    glClear(GL_COLOR_BUFFER_BIT);
    glLoadIdentity();

    drawSky();
    drawStars();
    drawMoon();
    drawClouds2();
    drawGround();

    for(int i = 0; i < MAX_TREES; i++) drawTree(&trees[i]);

    drawRiver();

    drawFighterShip(&ship);

    for(int i = 0; i < MAX_WINDMILLS; i++) drawWindmill(&windmills[i]);

    for(int i=0;i<3;i++) drawBuilding(&buildings[i]);

    drawBarbedWire(28,GY+6,430,GY+2);
    drawBarbedWire(28,GY+14,430,GY+10);
    drawSandbagWall(32,GY,14);
    drawSandbagWall(50,GY+13,12);
    for(int i=0;i<7;i++) drawBarricade(38+i*44,GY);

    if(!allBuildingsDestroyed){
        drawADLauncher(395,GY,SIDE_LEFT);
        drawADLauncher(440,GY,SIDE_LEFT);
        drawAttackLauncher(188,GY,.50f,SIDE_LEFT,.80f);
        drawAttackLauncher(282,GY,.46f,SIDE_LEFT,.80f);
    }

    {
        float tc=105,ty=(float)GY;
        float td=1.f,sc=.70f;
        float bx2=tc+td*36*sc,by2=ty+26*sc;
        registerTank(tc,ty,bx2,by2,SIDE_LEFT);
        drawTankAt(tc,ty+5,(int)td,sc,
                   (numTanks>0?tanks[0].damage:0),
                   (numTanks>0?tanks[0].destroyed:0));
    }

    for(int i=3;i<6;i++) drawBuilding(&buildings[i]);

    drawBarbedWire(770,GY+6,1172,GY+2);
    drawBarbedWire(770,GY+14,1172,GY+10);
    drawSandbagWall(772,GY,14);
    drawSandbagWall(790,GY+13,12);
    for(int i=0;i<7;i++) drawBarricade(778+i*48,GY);

    if(!allBuildingsDestroyed){
        drawADLauncher(760,GY,SIDE_RIGHT);
        drawADLauncher(715,GY,SIDE_RIGHT);
        drawAttackLauncher(910,GY,.50f,SIDE_RIGHT,.80f);
        drawAttackLauncher(1004,GY,.46f,SIDE_RIGHT,.80f);
    }

    {
        float tc=1095,ty=(float)GY;
        float td=-1.f,sc=.70f;
        float bx2=tc+td*36*sc,by2=ty+26*sc;
        registerTank(tc,ty,bx2,by2,SIDE_RIGHT);
        drawTankAt(tc,ty+5,0,sc,
                   (numTanks>1?tanks[1].damage:0),
                   (numTanks>1?tanks[1].destroyed:0));
    }

    for(int i=0;i<MAX_CIVILIANS;i++) drawCivilian(&civs[i]);

    drawAllAircraft();
    drawAllMissiles();
    drawParticles();
    drawHUD();
    glutSwapBuffers();
}

void timerCB(int v){
    (void)v;
    if(!paused){
        globalTime+=.016f;
        updateClouds();
        updateCivilians();
        updateAircraft();
        updateShip();
        updateWindmills();
        updateMissiles();
        updateParticles();
        updateBuildings();

        // One-shot: immediately clear all attack missiles when war just ended
        if(warJustEnded){
            clearAllAttackMissiles();
            warJustEnded = 0;
        }

        if(!allBuildingsDestroyed){
            leftMissileTimer+=.016f*gameSpeed;
            rightMissileTimer+=.016f*gameSpeed;
            if(leftMissileTimer>=leftMissileDelay){
                launchAttackMissile(SIDE_LEFT);
                leftMissileTimer=0;leftMissileDelay=randF(3.2f,5.8f);}
            if(rightMissileTimer>=rightMissileDelay){
                launchAttackMissile(SIDE_RIGHT);
                rightMissileTimer=0;rightMissileDelay=randF(3.f,5.5f);}

            tankTimerL+=.016f*gameSpeed;tankTimerR+=.016f*gameSpeed;
            if(tankTimerL>=tankDelayL){
                launchTankShell(SIDE_LEFT);
                tankTimerL=0;tankDelayL=randF(8,14);}
            if(tankTimerR>=tankDelayR){
                launchTankShell(SIDE_RIGHT);
                tankTimerR=0;tankDelayR=randF(9,15);}
        }
    }
    glutPostRedisplay();
    glutTimerFunc(16,timerCB,0);
}

void keyboard(unsigned char key,int x,int y){
    (void)x;(void)y;
    switch(key){
        case '1':gameSpeed=.5f;break;
        case '2':gameSpeed=1.f;break;
        case '3':gameSpeed=2.f;break;
        case 'p':case'P':paused=!paused;break;
        case 'r':case'R':resetScene();break;
        case 27:exit(0);}
}

int main(int argc,char**argv){
    glutInit(&argc,argv);
    glutInitDisplayMode(GLUT_DOUBLE|GLUT_RGB);
    glutInitWindowSize(W,H);
    glutInitWindowPosition(40,30);
    glutCreateWindow("War Simulation - Ship Boss Battle");
    glClearColor(.010f,.013f,.068f,1.f);
    glMatrixMode(GL_PROJECTION);glLoadIdentity();
    gluOrtho2D(0,W,0,H);
    glMatrixMode(GL_MODELVIEW);
    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA,GL_ONE_MINUS_SRC_ALPHA);
    glPointSize(1.4f);
    initScene();
    glutDisplayFunc(display);
    glutKeyboardFunc(keyboard);
    glutTimerFunc(0,timerCB,0);
    glutMainLoop();
    return 0;
}
