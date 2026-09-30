#include <GL/glut.h>
#include <cstdlib>
#include <iostream>
#include <string>
#include <cmath>
#include <ctime>
using namespace std;

int selectedGame=0; // 0=menu,1=snake,2=flappy,3=rock

// ---------- SNAKE ----------
#define S_WIDTH 40
#define S_HEIGHT 40
int snake_length=5,snakeX[400],snakeY[400];
int foodX,foodY;
int goldenX=-1,goldenY=-1;
int dir=1;
bool snakeGameOver=false;
int snakeScore=0;
int applesEaten=0;

// ---------- FLAPPY BIRD ----------
float birdY=0.3f, velocity=0.0f;
const float gravity=-0.002f;
struct Cactus { float x,y; float width,height; bool passed; };
const int NUM_CACTUS=6;
Cactus cactuses[NUM_CACTUS];
bool flappyGameOver=false;
int flappyScore=0;

// ---------- FALLING ROCK ----------
float playerX=0.0f;
struct Rock{ float x,y; float size; };
Rock rocks[1];
bool rockGameOver=false;
int rockScore=0;
int missedRocks=0;
const int maxMissed=1;

// ================= HELPERS ===================
void drawRect(float x,float y,float w=0.05f,float h=0.05f){
    glBegin(GL_QUADS);
    glVertex2f(x,y); glVertex2f(x+w,y);
    glVertex2f(x+w,y+h); glVertex2f(x,y+h);
    glEnd();
}

void drawCircle(float cx,float cy,float r,int seg=30){
    glBegin(GL_TRIANGLE_FAN);
    glVertex2f(cx,cy);
    for(int i=0;i<=seg;i++){
        float a=i*2.0f*3.141592f/seg;
        glVertex2f(cx+cos(a)*r,cy+sin(a)*r);
    }
    glEnd();
}

void drawText(float x,float y,const string &s){
    glRasterPos2f(x,y);
    for(char c:s) glutBitmapCharacter(GLUT_BITMAP_HELVETICA_18,c);
}

// ================= SNAKE ===================
void generateFood(){
    do { foodX=rand()%S_WIDTH; foodY=rand()%S_HEIGHT;}
    while((foodX==snakeX[0] && foodY==snakeY[0]) || (goldenX!=-1 && foodX==goldenX && foodY==goldenY));
}

void tryGenerateGolden(){
    if(applesEaten%2==0 && applesEaten!=0){
        do{ goldenX=rand()%S_WIDTH; goldenY=rand()%S_HEIGHT;}
        while((goldenX==foodX && goldenY==foodY) || (goldenX==snakeX[0] && goldenY==snakeY[0]));
    }
}

void drawSnake(){
    glColor3f(0,0.8f,0);
    for(int i=1;i<snake_length;i++) drawRect(snakeX[i]*0.05f-1.0f,snakeY[i]*0.05f-1.0f);
    glColor3f(0,1,0);
    drawRect(snakeX[0]*0.05f-1.0f,snakeY[0]*0.05f-1.0f);
}

void drawRedApple(){
    float x=foodX*0.05f-1.0f+0.025f;
    float y=foodY*0.05f-1.0f+0.025f;
    glColor3f(1,0,0); drawCircle(x,y,0.03f);
    glColor3f(0,0.5f,0); drawRect(x-0.003f,y+0.025f,0.006f,0.02f);
}

void drawGoldenApple(){
    if(goldenX==-1) return;
    float x=goldenX*0.05f-1.0f+0.025f;
    float y=goldenY*0.05f-1.0f+0.025f;
    glColor3f(1,0.84f,0); drawCircle(x,y,0.03f);
    glColor3f(1,1,0.4f); drawCircle(x,y,0.015f);
}

bool selfCollision(){
    for(int i=1;i<snake_length;i++)
        if(snakeX[0]==snakeX[i] && snakeY[0]==snakeY[i]) return true;
    return false;
}

void moveSnake(){
    for(int i=snake_length-1;i>0;i--){ snakeX[i]=snakeX[i-1]; snakeY[i]=snakeY[i-1]; }
    if(dir==0) snakeY[0]++; else if(dir==1) snakeX[0]++;
    else if(dir==2) snakeY[0]--; else if(dir==3) snakeX[0]--;

    if(snakeX[0]<0 || snakeX[0]>=S_WIDTH || snakeY[0]<0 || snakeY[0]>=S_HEIGHT){ snakeGameOver=true; return;}
    if(selfCollision()){ snakeGameOver=true; return;}

    if(snakeX[0]==foodX && snakeY[0]==foodY){
        snake_length++; snakeScore++; applesEaten++;
        generateFood(); tryGenerateGolden();
    }
    if(goldenX!=-1 && snakeX[0]==goldenX && snakeY[0]==goldenY){
        snake_length+=2; snakeScore+=5; goldenX=goldenY=-1;
    }
}

void initSnake(){
    snake_length=5; snakeScore=0; applesEaten=0; dir=1; snakeGameOver=false;
    for(int i=0;i<snake_length;i++){ snakeX[i]=20-i; snakeY[i]=20;}
    generateFood(); goldenX=goldenY=-1;
}

// ================= FLAPPY BIRD ===================
void drawFlappyBird(){
    float x=-0.8f; float y=birdY;
    // Body
    glColor3f(1,1,0); drawCircle(x,y,0.035f);
    // Beak
    glColor3f(1,0.6f,0);
    glBegin(GL_TRIANGLES);
        glVertex2f(x+0.035f,y);
        glVertex2f(x+0.055f,y+0.01f);
        glVertex2f(x+0.055f,y-0.01f);
    glEnd();
    // Eye
    glColor3f(0,0,0);
    drawCircle(x+0.015f,y+0.015f,0.005f);
    // Wings
    glColor3f(1,0.85f,0);
    glBegin(GL_TRIANGLES);
        float wingOffset=sin(glutGet(GLUT_ELAPSED_TIME)/100.0f)*0.015f;
        glVertex2f(x-0.035f,y);
        glVertex2f(x-0.055f,y+0.02f+wingOffset);
        glVertex2f(x-0.055f,y-0.02f-wingOffset);
    glEnd();
}

void drawCactus(Cactus c){
    glColor3f(0,0.7f,0);
    drawRect(c.x,c.y,c.width,c.height);
}

void moveFlappy(){
    if(flappyGameOver) return;
    velocity+=gravity; birdY+=velocity;
    float birdX=-0.8f;

    for(int i=0;i<NUM_CACTUS;i++){
        cactuses[i].x-=0.007f;

        // Collision
        if(birdX +0.035f>cactuses[i].x && birdX-0.035f<cactuses[i].x + cactuses[i].width){
            if(birdY+0.035f>cactuses[i].y && birdY-0.035f<cactuses[i].y+cactuses[i].height){
                flappyGameOver=true;
            }
        }

        // Score
        if(!cactuses[i].passed && birdX>cactuses[i].x+cactuses[i].width){
            flappyScore++;
            cactuses[i].passed=true;
        }

        // Reset cactus
        if(cactuses[i].x<-1.2f){
            cactuses[i].x=1.0f + (rand()%20)/10.0f;
            cactuses[i].y=-0.8f;
            cactuses[i].width=0.08f + (rand()%3)/50.0f;
            cactuses[i].height=0.3f + (rand()%4)/10.0f;
            cactuses[i].passed=false;
        }
    }

    if(birdY>0.9f) birdY=0.9f; // Ceiling limit
}

void initFlappy(){
    birdY=0.3f; velocity=0.0f; flappyScore=0; flappyGameOver=false;
    for(int i=0;i<NUM_CACTUS;i++){
        cactuses[i].x=1.0f + i*0.5f;
        cactuses[i].y=-0.8f;
        cactuses[i].width=0.08f + (rand()%3)/50.0f;
        cactuses[i].height=0.3f + (rand()%4)/10.0f;
        cactuses[i].passed=false;
    }
}

// ================= FALLING ROCK ===================
void drawPlayer(){
    float w=0.25f,h=0.1f;
    float x=playerX, y=-0.92f;
    glColor3f(0.1f,0.4f,0.8f);
    glBegin(GL_POLYGON);
    int seg=30;
    for(int i=0;i<=seg;i++){
        float angle=i*3.141592f/seg;
        glVertex2f(x + cos(angle)*w/2, y + sin(angle)*h);
    }
    for(int i=0;i<=seg;i++){
        float angle=i*3.141592f/seg;
        glVertex2f(x - cos(angle)*w/2, y + sin(angle)*h);
    }
    glEnd();
}

void drawRock(Rock r){
    glColor3f(0.5f,0.25f,0.1f);
    glBegin(GL_POLYGON);
    int points=6+rand()%5;
    for(int i=0;i<points;i++){
        float angle=i*2.0f*3.141592f/points;
        float radius=r.size + (rand()%10)/200.0f;
        glVertex2f(r.x + cos(angle)*radius, r.y + sin(angle)*radius);
    }
    glEnd();
}

void moveRock(){
    rocks[0].y-=0.012f;
    if(rocks[0].y<-1.0f){
        missedRocks++;
        if(missedRocks>=maxMissed) rockGameOver=true;
        rocks[0].y=1.1f;
        rocks[0].x=(rand()%180-90)/100.0f;
    }
    float dx=rocks[0].x-playerX, dy=rocks[0].y+0.85f;
    if(dx*dx + dy*dy<0.08f*0.08f){
        rockScore++;
        rocks[0].y=1.1f;
        rocks[0].x=(rand()%180-90)/100.0f;
    }
}

void initRock(){
    playerX=0.0f; rockScore=0; rockGameOver=false; missedRocks=0;
    rocks[0].x=(rand()%180-90)/100.0f; rocks[0].y=1.0f;
    rocks[0].size=0.05f;
}

// ================= DISPLAY ===================
void displayMenu(){
    glClearColor(0.05f,0.05f,0.15f,1); glClear(GL_COLOR_BUFFER_BIT);
    glColor3f(1,0.9f,0.7f); drawText(-0.5f,0.6f,"MINI ARCADE");
    glColor3f(0.3f,1,0.3f); drawText(-0.4f,0.3f,"1 - Snake");
    glColor3f(0.3f,0.8f,1); drawText(-0.4f,0.1f,"2 - Flappy Bird");
    glColor3f(1,0.4f,0.4f); drawText(-0.4f,-0.1f,"3 - Falling Rock");
    glutSwapBuffers();
}

void display(){
    if(selectedGame==0){ displayMenu(); return; }
    if(selectedGame==1) glClearColor(0,0.1f,0,1);
    else if(selectedGame==2) glClearColor(0.5f,0.8f,1.0f,1);
    else if(selectedGame==3) glClearColor(0.2f,0.2f,0.3f,1);
    glClear(GL_COLOR_BUFFER_BIT);

    if(selectedGame==1){
        drawSnake(); drawRedApple(); drawGoldenApple();
        glColor3f(1,1,1); drawText(-0.95f,0.9f,"Score: "+to_string(snakeScore));
        if(snakeGameOver){ drawText(-0.4f,0,"GAME OVER - Press R"); selectedGame=0; }
    }else if(selectedGame==2){
        for(int i=0;i<NUM_CACTUS;i++) drawCactus(cactuses[i]);
        drawFlappyBird();
        glColor3f(1,1,1); drawText(-0.95f,0.9f,"Score: "+to_string(flappyScore));
        if(flappyGameOver){ drawText(-0.4f,0,"GAME OVER - Press R"); selectedGame=0; }
    }else if(selectedGame==3){
        drawPlayer(); drawRock(rocks[0]);
        glColor3f(1,1,1); drawText(-0.95f,0.9f,"Score: "+to_string(rockScore));
        if(rockGameOver){ drawText(-0.4f,0,"GAME OVER - Press R"); selectedGame=0; }
    }
    glutSwapBuffers();
}

// ================= INPUT & TIMER ===================
void timer(int=0){
    if(selectedGame==1 && !snakeGameOver) moveSnake();
    if(selectedGame==2) moveFlappy();
    if(selectedGame==3 && !rockGameOver) moveRock();
    glutPostRedisplay();
    glutTimerFunc(100,timer,0); // medium snake speed
}

void keyboard(unsigned char key,int,int){
    if(selectedGame==0){
        if(key=='1'){ selectedGame=1; initSnake();}
        if(key=='2'){ selectedGame=2; initFlappy();}
        if(key=='3'){ selectedGame=3; initRock();}
    }
    if(key==' ' && selectedGame==2 && !flappyGameOver) velocity=0.025f;
    if(key=='r'||key=='R'){
        if(selectedGame==1) initSnake();
        else if(selectedGame==2) initFlappy();
        else if(selectedGame==3) initRock();
    }
}

void special(int key,int,int){
    if(selectedGame==1 && !snakeGameOver){
        if(key==GLUT_KEY_UP && dir!=2) dir=0;
        if(key==GLUT_KEY_RIGHT && dir!=3) dir=1;
        if(key==GLUT_KEY_DOWN && dir!=0) dir=2;
        if(key==GLUT_KEY_LEFT && dir!=1) dir=3;
    }
    if(selectedGame==3 && !rockGameOver){
        if(key==GLUT_KEY_LEFT) playerX-=0.1f;
        if(key==GLUT_KEY_RIGHT) playerX+=0.1f;
        if(playerX<-0.9f) playerX=-0.9f;
        if(playerX>0.9f) playerX=0.9f;
    }
}

// ================= MAIN ===================
int main(int argc,char** argv){
    srand(time(0));
    glutInit(&argc,argv);
#ifdef GLUT_MULTISAMPLE
    glutInitDisplayMode(GLUT_DOUBLE|GLUT_RGBA|GLUT_MULTISAMPLE);
#else
    glutInitDisplayMode(GLUT_DOUBLE|GLUT_RGBA);
#endif
    glutInitWindowSize(800,600);
    glutCreateWindow("Mini Arcade - Snake / Flappy / Falling Rock");
#ifdef GL_MULTISAMPLE
    glEnable(GL_MULTISAMPLE);
#endif
    gluOrtho2D(-1,1,-1,1);

    glutDisplayFunc(display);
    glutKeyboardFunc(keyboard);
    glutSpecialFunc(special);
    glutTimerFunc(100,timer,0); // snake medium speed

    glutMainLoop();
    return 0;
}
