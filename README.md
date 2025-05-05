#include <GL/glut.h>
#include <math.h>
#include <stdio.h>
#include <stdlib.h>
#include <cstdlib>
#include <time.h>


#define PI 3.141592653589793 // Pi de tinh goc
#define MAX_SPEED 2.0f // Toc do toi da
#define DECELERATION 0.001f // Giam toc tren mat dat
#define THRUST 0.01f // Luc day
#define PITCH_RATE 8.0f // Toc do quay cao do
#define YAW_RATE 2.0f // Toc do quay huong
#define ROLL_RATE 2.0f // Toc do quay 
#define MAX_CHARGE 1.0f // Nang luong toi da
#define PARTICLE_COUNT 100 // So hat hieu ung
#define AFTERBURNER_COUNT 500 // Hat dong co
#define WING_TRAIL_COUNT 100
#define CLOUD_COUNT 80 // So dam may 
#define TERRAIN_SCALE 50.0f // Ti le dia hinh
#define GROUND_SPEED 0.2f // Toc do 
#define GROUND_THRESHOLD 0.6f // banh xe
#define MAX_ALTITUDE 100.0f // Do cao toi da
#define WORLD_SIZE 100000.0f // Kich thuoc the gioi 3D
#define TREE_COUNT 100 // So cay
#define MAX_ENEMIES 3 // So vat can mo phong trong rada
#define RADAR_RANGE 100.0f // Pham vi radar

// Bien toan cuc
GLfloat radians(GLfloat a);
float angleX = 0.0f, angleY = 0.0f; 
float xpos = 0.0f, ypos = 0.5f, zpos = 0.0f; // Vi tri may bay
float speed = 0.0f, direction = 0.0f; // Toc do, huong
float camx = 0.0f, camy = 2.0f, camz = 5.0f; // Vi tri camera
float charge = 0.0f; // Nang luong nap
int isCharging = 0, isThrusting = 0, isAutoFlying = 0; // Trang thai nap, day, tu dong
int isSunny = 1, isCockpitView = 0, gearDown = 1, isWireframe = 0; //  goc nhin, banh xe

// Cau truc cho hat focal
typedef struct
{
    float x, y, z; // Vi tri
    float vx, vy, vz; // Van toc
    float life; 
    int isFire; 
} Particle;
Particle boosters[PARTICLE_COUNT], afterburner[AFTERBURNER_COUNT], wingTrails[WING_TRAIL_COUNT], explosion[PARTICLE_COUNT], clouds[CLOUD_COUNT];

// Cau truc cho cay
typedef struct
{
    float x, z; // Vi tri
} Tree;
Tree trees[TREE_COUNT];

// Cau truc cho vat can
typedef struct
{
    float x, z; // Vi tri
    float life; // Thoi gian xuat hien
    int active; // Hoat dong
} Enemy;
Enemy enemies[MAX_ENEMIES];

// Tinh do cao dia hinh
float terrainHeight(float x, float z)
{
    // Tao dia hinh len xuong bang sin, cos
    float x_mod = fmod(x + TERRAIN_SCALE * 2, TERRAIN_SCALE * 2) - TERRAIN_SCALE;
    float z_mod = fmod(z + TERRAIN_SCALE * 2, TERRAIN_SCALE * 2) - TERRAIN_SCALE;
    return 0.5f + 0.5f * sin(x_mod / 10.0f) * cos(z_mod / 10.0f);
}

// Chuyen toa do cuc bo sang toa do the gioi
void localToWorld(float localX, float localY, float localZ, float* worldX, float* worldY, float* worldZ)
{
    // Dung goc huong, cao do, lan de tinh toan
    float radDir = radians(direction);
    float radPitch = radians(angleX);
    float radRoll = radians(angleY);

    float tempX = localX * cos(radDir) + localZ * sin(radDir);
    float tempZ = -localX * sin(radDir) + localZ * cos(radDir);
    float tempY = localY;

    float finalY = tempY * cos(radPitch) - tempZ * sin(radPitch);
    float finalZ = tempY * sin(radPitch) + tempZ * cos(radPitch);
    float finalX = tempX;

    float rolledX = finalX * cos(radRoll) - finalY * sin(radRoll);
    float rolledY = finalX * sin(radRoll) + finalY * cos(radPitch);
    *worldX = xpos + rolledX;
    *worldY = ypos + rolledY;
    *worldZ = zpos + finalZ;
}

// Khoi tao hat cho hieu ung
void initParticles(Particle* particles, int count, float x, float y, float z, int isExplosion, int isWingTrail)
{
    // Thiet lap vi tri, van toc, thoi gian xuat hien cho hat
    for (int i = 0; i < count; i++)
    {
        particles[i].x = x;
        particles[i].y = y;
        particles[i].z = z;
        if (isExplosion)
        {
            particles[i].vx = ((float)rand() / RAND_MAX - 0.5f) * 0.5f;
            particles[i].vy = ((float)rand() / RAND_MAX - 0.5f) * 0.5f;
            particles[i].vz = ((float)rand() / RAND_MAX - 0.5f) * 0.5f;
            particles[i].isFire = 1;
            particles[i].life = 1.0f;
        }
        else if (isWingTrail)
        {
            float radDir = radians(direction);
            float baseVx = -0.1f * cos(radDir);
            float baseVz = 0.1f * sin(radDir);
            particles[i].vx = baseVx + ((float)rand() / RAND_MAX - 0.5f) * 0.1f;
            particles[i].vy = ((float)rand() / RAND_MAX - 0.5f) * 0.1f;
            particles[i].vz = baseVz + ((float)rand() / RAND_MAX - 0.5f) * 0.1f;
            particles[i].isFire = ((float)rand() / RAND_MAX < 0.2f) ? 1 : 0;
            particles[i].life = 2.0f;
        }
        else
        {
            float radDir = radians(direction);
            float baseVx = -0.3f * cos(radDir);
            float baseVz = 0.3f * sin(radDir);
            particles[i].vx = baseVx + ((float)rand() / RAND_MAX - 0.5f) * 0.2f;
            particles[i].vy = ((float)rand() / RAND_MAX - 0.5f) * 0.2f;
            particles[i].vz = baseVz + ((float)rand() / RAND_MAX - 0.5f) * 0.2f;
            particles[i].isFire = (particles == afterburner && (float)rand() / RAND_MAX < 0.8f) ? 1 : 0;
            particles[i].life = 3.0f;
        }
    }
}

// Khoi tao dam may
void initClouds()
{
    // Dat dam may ngau nhien quanh may bay voi do cao cao  va chuyen dong
    for (int i = 0; i < CLOUD_COUNT; i++)
    {
        clouds[i].x = xpos + ((float)rand() / RAND_MAX - 0.5f) * 300.0f;
        clouds[i].y = 30.0f + ((float)rand() / RAND_MAX) * 30.0f; // Do cao tu 15.0 den 30.0
        clouds[i].z = zpos + ((float)rand() / RAND_MAX - 0.5f) * 300.0f;
        clouds[i].vx = ((float)rand() / RAND_MAX - 0.5f) * 0.05f; // Van toc nho de troi
        clouds[i].vy = 0.0f;
        clouds[i].vz = ((float)rand() / RAND_MAX - 0.5f) * 0.05f;
        clouds[i].life = 1.0f;
        clouds[i].isFire = 0;
    }
}

// Khoi tao cay
void initTrees()
{
    // Dat cay ngau nhien
    for (int i = 0; i < TREE_COUNT; i++)
    {
        do
        {
            trees[i].x = xpos + ((float)rand() / RAND_MAX - 0.5f) * 200.0f;
            trees[i].z = zpos + ((float)rand() / RAND_MAX - 0.5f) * 200.0f;
        } while (fabs(trees[i].x) < 10.0f && fabs(trees[i].z) < 2.0f);
    }
}

// Khoi tao vat can
void initEnemies()
{
    // Thiet lap vat can ban dau khong hoat dong
    for (int i = 0; i < MAX_ENEMIES; i++)
    {
        enemies[i].x = 0.0f;
        enemies[i].z = 0.0f;
        enemies[i].life = 0.0f;
        enemies[i].active = 0;
    }
}

//  radar
void Radar()
{
    // Cap nhat vat can, tao vat can moi ngau nhien
    for (int i = 0; i < MAX_ENEMIES; i++)
    {
        if (enemies[i].active)
        {
            enemies[i].life -= 0.016f;
            if (enemies[i].life <= 0.0f)
            {
                enemies[i].active = 0;
            }
        }
    }

    for (int i = 0; i < MAX_ENEMIES; i++)
    {
        if (!enemies[i].active && (float)rand() / RAND_MAX < 0.02f)
        {
            float angle = (float)rand() / RAND_MAX * 2.0f * PI;
            float distance = (float)rand() / RAND_MAX * RADAR_RANGE;
            enemies[i].x = distance * cos(angle);
            enemies[i].z = distance * sin(angle);
            enemies[i].life = 3.0f + (float)rand() / RAND_MAX * 2.0f;
            enemies[i].active = 1;
        }
    }
}

// Ve cay
void drawTrees()
{
    // Ve cay voi than (khoi lap phuong) va tan (hinh non)
    for (int i = 0; i < TREE_COUNT; i++)
    {
        float y = terrainHeight(trees[i].x, trees[i].z);
        glPushMatrix();
        glTranslatef(trees[i].x, y, trees[i].z);
        glColor3f(0.6f, 0.3f, 0.0f);
        glPushMatrix();
        glScalef(0.1f, 0.5f, 0.1f);
        glutSolidCube(1.0f);
        glPopMatrix();
        glColor3f(0.0f, 0.6f, 0.0f);
        glPushMatrix();
        glTranslatef(0.0f, 0.5f, 0.0f);
        glRotatef(-90.0f, 1.0f, 0.0f, 0.0f);
        glutSolidCone(0.3f, 0.7f, 10, 10);
        glPopMatrix();
        glPopMatrix();
    }
}

// Di chuyen hat
void moveParticles(Particle* particles, int count, float dt, int isExplosion)
{
    for (int i = 0; i < count; i++)
    {
        if (particles[i].life > 0.0f)
        {
            particles[i].x += particles[i].vx * dt;
            particles[i].y += particles[i].vy * dt;
            particles[i].z += particles[i].vz * dt;
            particles[i].life -= dt * (isExplosion ? 2.0f : 0.3f);
        }
    }
}

// Di chuyen dam may
void moveClouds()
{
    // Cap nhat vi tri dam may va tai tao neu qua xa
    for (int i = 0; i < CLOUD_COUNT; i++)
    {
        if (clouds[i].life > 0.0f)
        {
            clouds[i].x += clouds[i].vx * 0.016f;
            clouds[i].z += clouds[i].vz * 0.016f;
            // Them hieu ung bong benh
            clouds[i].y += 0.02f * sin(glutGet(GLUT_ELAPSED_TIME) * 0.001f + i);
            float dx = clouds[i].x - xpos;
            float dz = clouds[i].z - zpos;
            if (dx * dx + dz * dz > 300.0f * 300.0f)
            {
                clouds[i].x = xpos + ((float)rand() / RAND_MAX - 0.5f) * 300.0f;
                clouds[i].y = 15.0f + ((float)rand() / RAND_MAX) * 15.0f;
                clouds[i].z = zpos + ((float)rand() / RAND_MAX - 0.5f) * 300.0f;
                clouds[i].vx = ((float)rand() / RAND_MAX - 0.5f) * 0.05f;
                clouds[i].vz = ((float)rand() / RAND_MAX - 0.5f) * 0.05f;
                clouds[i].life = 1.0f;
            }
        }
    }
}

// Ve hat
void drawParticles(Particle* particles, int count, int isFireDefault, int isWingTrail)
{
    // Ve hat voi mau sac va do trong suot
    glEnable(GL_BLEND);
    if ((particles == afterburner || isWingTrail) && isFireDefault)
    {
        glBlendFunc(GL_SRC_ALPHA, GL_ONE);
    }
    else
    {
        glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    }
    for (int i = 0; i < count; i++)
    {
        if (particles[i].life > 0.0f)
        {
            glPointSize(isWingTrail ? 4.0f : (particles == afterburner ? 8.0f : 15.0f));
            glBegin(GL_POINTS);
            if (particles == afterburner || isWingTrail)
            {
                if (particles[i].isFire)
                {
                    float intensity = particles[i].life;
                    glColor4f(1.0f, isWingTrail ? 0.9f : 0.7f + intensity * 0.3f, 
                              isWingTrail ? 0.8f : 0.2f * intensity, 
                              isWingTrail ? 0.7f * intensity : 0.9f * intensity);
                }
                else
                {
                    glColor4f(0.3f, 0.3f, 0.3f, particles[i].life * (isWingTrail ? 0.8f : 0.6f));
                }
            }
            else if (isFireDefault)
            {
                glColor4f(1.0f, 0.5f, 0.0f, particles[i].life);
            }
            else
            {
                glColor4f(0.5f, 0.5f, 0.5f, particles[i].life);
            }
            glVertex3f(particles[i].x, particles[i].y, particles[i].z);
            glEnd();
        }
    }
    glDisable(GL_BLEND);
    glPointSize(1.0f);
}

// Ve dam may
void drawClouds()
{
    // Ve dam may bang nhieu hinh cau voi do trong suot bien doi
    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    for (int i = 0; i < CLOUD_COUNT; i++)
    {
        if (clouds[i].life > 0.0f)
        {
            glPushMatrix();
            glTranslatef(clouds[i].x, clouds[i].y, clouds[i].z);
            // Ve nhieu hinh cau de tao hinh dang mem mai
            float alpha = 0.4f + 0.2f * sin(glutGet(GLUT_ELAPSED_TIME) * 0.001f + i); // Do trong suot thay doi
            glColor4f(1.0f, 1.0f, 1.0f, alpha);
            glutSolidSphere(2.0f, 10, 10); // Hinh cau chinh
            glTranslatef(2.0f, 0.0f, 0.0f);
            glutSolidSphere(2.0f, 10, 10); // Hinh cau phu
            glTranslatef(-4.0f, 0.0f, 0.0f);
            glutSolidSphere(2.0f, 10, 10); // Hinh cau phu
            glPopMatrix();
        }
    }
    glDisable(GL_BLEND);
}

// Dat lai mo phong
void resetSimulation()
{
    // Dat lai vi tri, trang thai ban dau
    xpos = -8.0f;
    ypos = terrainHeight(-8.0f, 0.0f) + 0.2f;
    zpos = 0.0f;
    speed = 0.0f;
    direction = 0.0f;
    angleX = 0.0f;
    angleY = 0.0f;
    gearDown = 1;
    charge = 0.0f;
    isCharging = 0;
    isThrusting = 0;
    isAutoFlying = 0;
    isSunny = 1;
    isCockpitView = 0;
    isWireframe = 0;
    camx = 0.0f;
    camy = 2.0f;
    camz = 5.0f;
    initParticles(boosters, PARTICLE_COUNT, 0.0f, 0.0f, 0.0f, 0, 0);
    initParticles(afterburner, AFTERBURNER_COUNT, 0.0f, 0.0f, 0.0f, 0, 0);
    initParticles(wingTrails, WING_TRAIL_COUNT, 0.0f, 0.0f, 0.0f, 0, 1);
    initParticles(explosion, PARTICLE_COUNT, 0.0f, 0.0f, 0.0f, 1, 0);
    initClouds();
    initTrees();
    initEnemies();
    printf("Da dat lai trang thai ban dau tren duong bang.\n");
}

// Ve may bay
void plane()
{
    // Ve may bay voi cac hinh khoi co ban
    if (isWireframe)
    {
        glPolygonMode(GL_FRONT_AND_BACK, GL_LINE);
    }
    else
    {
        glPolygonMode(GL_FRONT_AND_BACK, GL_FILL);
    }

    glColor3d(1.0, 0.0, 0.0);
    glPushMatrix();
    glTranslated(0, 0, 0);
    glScaled(3, 0.4, 0.5);
    glutSolidSphere(1, 30, 30);
    glPopMatrix();

    glColor3d(0, 0, 0);
    glPushMatrix();
    glTranslated(1.7, 0.1, 0);
    glScaled(1.5, 0.7, 0.8);
    glRotated(40, 0, 1, 0);
    glutSolidSphere(0.45, 30, 30);
    glPopMatrix();

    glColor3f(1.0f, 1.0f, 0.0f);
    glPushMatrix();
    glTranslated(0, 0, 1.2);
    glRotated(-50, 0, 1, 0);
    glScaled(0.7, 0.1, 3);
    glRotated(25, 0, 1, 0);
    glutSolidCube(1);
    glPopMatrix();

    glPushMatrix();
    glTranslated(-0.3, -0.15, 1.5);
    glRotated(90, 0, 1, 0);
    glScaled(0.1, 0.1, 0.9);
    glutSolidTorus(0.5, 0.5, 50, 50);
    glPopMatrix();
    glPushMatrix();
    glTranslated(0.2, -0.15, 0.9);
    glRotated(90, 0, 1, 0);
    glScaled(0.1, 0.1, 0.9);
    glutSolidTorus(0.5, 0.5, 50, 50);
    glPopMatrix();

    glColor3f(1.0f, 1.0f, 0.0f);
    glPushMatrix();
    glTranslated(0, 0, -1.2);
    glRotated(50, 0, 1, 0);
    glScaled(0.7, 0.1, 3);
    glRotated(-25, 0, 1, 0);
    glutSolidCube(1);
    glPopMatrix();

    glPushMatrix();
    glTranslated(-0.3, -0.15, -1.5);
    glRotated(90, 0, 1, 0);
    glScaled(0.1, 0.1, 0.9);
    glutSolidTorus(0.5, 0.5, 50, 50);
    glPopMatrix();
    glPushMatrix();
    glTranslated(0.2, -0.15, -0.9);
    glRotated(90, 0, 1, 0);
    glScaled(0.1, 0.1, 0.9);
    glutSolidTorus(0.5, 0.5, 50, 50);
    glPopMatrix();

    glColor3f(1.0f, 1.0f, 0.0f);
    glPushMatrix();
    glTranslated(-2.8, 0, 0);
    glScaled(0.8, 0.5, 0.3);
    glPushMatrix();
    glTranslated(0.4, 0, 1.5);
    glRotated(-30, 0, 1, 0);
    glScaled(0.7, 0.1, 3);
    glRotated(10, 0, 1, 0);
    glutSolidCube(1);
    glPopMatrix();
    glPushMatrix();
    glTranslated(0.4, 0, -1.5);
    glRotated(30, 0, 1, 0);
    glScaled(0.7, 0.1, 3);
    glRotated(-10, 0, 1, 0);
    glutSolidCube(1);
    glPopMatrix();
    glPopMatrix();

    glColor3f(1.0f, 1.0f, 0.0f);
    glPushMatrix();
    glTranslated(-2.7, 0.5, 0);
    glRotated(45, 0, 0, 1);
    glScaled(0.8, 2, 0.1);
    glRotated(-20, 0, 0, 1);
    glutSolidCube(0.5);
    glPopMatrix();

    if (gearDown)
    {
        glColor3f(0.1f, 0.1f, 0.1f);
        glPushMatrix();
        glTranslated(1.5, -0.4, 0.0);
        glScaled(0.1, 0.3, 0.1);
        glutSolidCube(1);
        glPopMatrix();
        glColor3f(0.05f, 0.05f, 0.05f);
        glPushMatrix();
        glTranslated(1.5, -0.55, 0.0);
        glRotated(90, 1, 0, 0);
        glutSolidTorus(0.05, 0.15, 10, 20);
        glPopMatrix();

        glColor3f(0.1f, 0.1f, 0.1f);
        glPushMatrix();
        glTranslated(0.0, -0.4, 0.8);
        glScaled(0.1, 0.3, 0.1);
        
        glutSolidCube(1);
        glPopMatrix();
        glColor3f(0.05f, 0.4, 0.05f);
        glPushMatrix();
        glTranslated(0.0, -0.55, 0.8);
        glRotated(90, 1, 0, 0);
        glutSolidTorus(0.05, 0.15, 10, 20);
        glPopMatrix();

        glColor3f(0.1f, 0.1f, 0.1f);
        glPushMatrix();
        glTranslated(0.0, -0.4, -0.8);
        glScaled(0.1, 0.3, 0.1);
        glutSolidCube(1);
        glPopMatrix();
        glColor3f(0.05f, 0.05f, 0.05f);
        glPushMatrix();
        glTranslated(0.0, -0.55, -0.8);
        glRotated(90, 1, 0, 0);
        glutSolidTorus(0.05, 0.15, 10, 20);
        glPopMatrix();
    }

    glPolygonMode(GL_FRONT_AND_BACK, GL_FILL);
}

// Ve dia hinh, cay
void drawTerrain()
{
    glColor3f(0.3f, 0.2f, 0.1f);
    glBegin(GL_QUADS);
    float view_range = 100.0f;
    for (float x = xpos - view_range; x < xpos + view_range; x += 1.0f)
    {
        for (float z = zpos - view_range; z < zpos + view_range; z += 1.0f)
        {
            float h1 = terrainHeight(x, z);
            float h2 = terrainHeight(x + 1.0f, z);
            float h3 = terrainHeight(x + 1.0f, z + 1.0f);
            float h4 = terrainHeight(x, z + 1.0f);
            glVertex3f(x, h1, z);
            glVertex3f(x + 1.0f, h2, z);
            glVertex3f(x + 1.0f, h3, z + 1.0f);
            glVertex3f(x, h4, z + 1.0f);
        }
    }
    glEnd();
    
    drawTrees();
}

// Ve luoi tham chieu
void landmarks()
{
    GLfloat i;
    glColor3f(0.0f, 1.0f, 0.0f);
    glBegin(GL_LINES);
    float grid_range = 10000.0f;
    for (i = xpos - grid_range; i <= xpos + grid_range; i += 1.0f)
    {
        glVertex3f(xpos - grid_range, -0.5f, i);
        glVertex3f(xpos + grid_range, -0.5f, i);
    }
    for (i = zpos - grid_range; i <= zpos + grid_range; i += 1.0f)
    {
        glVertex3f(i, -0.5f, zpos - grid_range);
        glVertex3f(i, -0.5f, zpos + grid_range);
    }
    glEnd();
}

GLfloat Abs(GLfloat a)
{
    return (a < 0.0f) ? -a : a;
}

GLfloat radians(GLfloat a)
{
    return a * PI / 180.0f;
}

// Ve thanh do cao
void drawAltitudeBar()
{
    glMatrixMode(GL_PROJECTION);
    glPushMatrix();
    glLoadIdentity();
    gluOrtho2D(0, 1200, 0, 600);
    glMatrixMode(GL_MODELVIEW);
    glPushMatrix();
    glLoadIdentity();

    glDisable(GL_LIGHTING);
    glDisable(GL_DEPTH_TEST);

    glColor3f(0.2f, 0.2f, 0.2f);
    glBegin(GL_QUADS);
    glVertex2f(20, 50);
    glVertex2f(80, 50);
    glVertex2f(80, 550);
    glVertex2f(20, 550);
    glEnd();

    float maxDisplayAltitude = 0.10f;
    float altitudeKm = ypos / 1000.0f;
    float altitudeRatio = (altitudeKm < 0.0f ? 0.0f : (altitudeKm > maxDisplayAltitude ? 1.0f : altitudeKm / maxDisplayAltitude));
    float barHeight = 500 * altitudeRatio;
    if (barHeight < 0.0f) barHeight = 0.0f;
    if (barHeight > 500.0f) barHeight = 500.0f;

    glColor3f(1.0f, 1.0f, 0.0f);
    glBegin(GL_QUADS);
    glVertex2f(20, 50);
    glVertex2f(80, 50);
    glVertex2f(80, 50 + barHeight);
    glVertex2f(20, 50 + barHeight);
    glEnd();

    glLineWidth(3.0f);
    glColor3f(1.0f, 1.0f, 1.0f);
    glBegin(GL_LINE_LOOP);
    glVertex2f(20, 50);
    glVertex2f(80, 50);
    glVertex2f(80, 550);
    glVertex2f(20, 550);
    glEnd();
    glLineWidth(1.0f);

    glColor3f(1.0f, 1.0f, 1.0f);
    char label[10];
    for (float alt = 0.0f; alt <= 0.10f; alt += 0.025f)
    {
        float yPos = 50 + (alt / maxDisplayAltitude) * 500;
        glRasterPos2f(85, yPos - 5);
        snprintf(label, sizeof(label), "%.2f km", alt * 150.0f);
        for (char* c = label; *c != '\0'; c++)
        {
            glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
        }
    }

    glEnable(GL_LIGHTING);
    glEnable(GL_DEPTH_TEST);

    glPopMatrix();
    glMatrixMode(GL_PROJECTION);
    glPopMatrix();
    glMatrixMode(GL_MODELVIEW);
}

// Ve la ban
void drawCompass()
{
    glMatrixMode(GL_PROJECTION);
    glPushMatrix();
    glLoadIdentity();
    gluOrtho2D(0, 1200, 0, 600);
    glMatrixMode(GL_MODELVIEW);
    glPushMatrix();
    glLoadIdentity();

    glDisable(GL_LIGHTING);
    glDisable(GL_DEPTH_TEST);

    float centerX = 1100.0f;
    float centerY = 100.0f;
    float radius = 40.0f;

    glColor3f(0.2f, 0.2f, 0.2f);
    glBegin(GL_POLYGON);
    for (int i = 0; i < 360; i += 5)
    {
        float rad = radians(i);
        glVertex2f(centerX + radius * cos(rad), centerY + radius * sin(rad));
    }
    glEnd();

    glLineWidth(2.0f);
    glColor3f(1.0f, 1.0f, 1.0f);
    glBegin(GL_LINE_LOOP);
    for (int i = 0; i < 360; i += 5)
    {
        float rad = radians(i);
        glVertex2f(centerX + radius * cos(rad), centerY + radius * sin(rad));
    }
    glEnd();
    glLineWidth(1.0f);

    glColor3f(1.0f, 1.0f, 1.0f);
    const char* directions[] = {"N", "E", "S", "W"};
    float labelAngles[] = {0.0f, 90.0f, 180.0f, 270.0f};
    for (int i = 0; i < 4; i++)
    {
        float rad = radians(labelAngles[i]);
        float labelX = centerX + (radius + 10.0f) * cos(rad);
        float labelY = centerY + (radius + 10.0f) * sin(rad);
        glRasterPos2f(labelX - 5.0f, labelY - 5.0f);
        for (const char* c = directions[i]; *c != '\0'; c++)
        {
            glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
        }
    }

    glPushMatrix();
    glTranslatef(centerX, centerY, 0.0f);
    glRotatef(-direction, 0.0f, 0.0f, 1.0f);
    glColor3f(1.0f, 0.0f, 0.0f);
    glBegin(GL_LINES);
    glVertex2f(0.0f, -radius * 0.8f);
    glVertex2f(0.0f, radius * 0.8f);
    glEnd();
    glBegin(GL_TRIANGLES);
    glVertex2f(-5.0f, radius * 0.6f);
    glVertex2f(5.0f, radius * 0.6f);
    glVertex2f(0.0f, radius * 0.8f);
    glEnd();
    glPopMatrix();

    glEnable(GL_LIGHTING);
    glEnable(GL_DEPTH_TEST);

    glPopMatrix();
    glMatrixMode(GL_PROJECTION);
    glPopMatrix();
    glMatrixMode(GL_MODELVIEW);
}

// Ve radar
void drawRadar()
{
    glMatrixMode(GL_PROJECTION);
    glPushMatrix();
    glLoadIdentity();
    gluOrtho2D(0, 1200, 0, 600);
    glMatrixMode(GL_MODELVIEW);
    glPushMatrix();
    glLoadIdentity();

    glDisable(GL_LIGHTING);
    glDisable(GL_DEPTH_TEST);

    float centerX = 1000.0f;
    float centerY = 100.0f;
    float radius = 40.0f;
    float scale = radius / RADAR_RANGE;

    glColor3f(0.2f, 0.2f, 0.2f);
    glBegin(GL_POLYGON);
    for (int i = 0; i < 360; i += 5)
    {
        float rad = radians(i);
        glVertex2f(centerX + radius * cos(rad), centerY + radius * sin(rad));
    }
    glEnd();

    glLineWidth(2.0f);
    glColor3f(1.0f, 1.0f, 1.0f);
    glBegin(GL_LINE_LOOP);
    for (int i = 0; i < 360; i += 5)
    {
        float rad = radians(i);
        glVertex2f(centerX + radius * cos(rad), centerY + radius * sin(rad));
    }
    glEnd();
    glLineWidth(1.0f);

    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    float sweepAngle = fmod(glutGet(GLUT_ELAPSED_TIME) / 4000.0f * 360.0f, 360.0f);
    glColor4f(0.0f, 1.0f, 0.0f, 0.3f);
    glBegin(GL_TRIANGLE_FAN);
    glVertex2f(centerX, centerY);
    for (int i = 0; i <= 30; i++)
    {
        float angle = radians(sweepAngle + i);
        glVertex2f(centerX + radius * cos(angle), centerY + radius * sin(angle));
    }
    glEnd();
    glDisable(GL_BLEND);

    glPointSize(5.0f);
    glColor3f(1.0f, 0.0f, 0.0f);
    glBegin(GL_POINTS);
    for (int i = 0; i < MAX_ENEMIES; i++)
    {
        if (enemies[i].active)
        {
            float screenX = centerX + enemies[i].x * scale;
            float screenZ = centerY + enemies[i].z * scale;
            glVertex2f(screenX, screenZ);
        }
    }
    glEnd();
    glPointSize(1.0f);

    glPushMatrix();
    glTranslatef(centerX, centerY, 0.0f);
    glRotatef(-direction, 0.0f, 0.0f, 1.0f);
    glColor3f(1.0f, 1.0f, 1.0f);
    glBegin(GL_TRIANGLES);
    glVertex2f(0.0f, 6.0f); // S?a t? glVertex pinch thành glVertex2f
    glVertex2f(-3.0f, -3.0f);
    glVertex2f(3.0f, -3.0f);
    glEnd();
    glPopMatrix();

    glEnable(GL_LIGHTING);
    glEnable(GL_DEPTH_TEST);

    glPopMatrix();
    glMatrixMode(GL_PROJECTION);
    glPopMatrix();
    glMatrixMode(GL_MODELVIEW);
}

// Cap nhat canh
void advanceScene()
{
    // Cap nhat vi tri, toc do, hat, va cham
    GLfloat xDelta, zDelta, yDelta;

    float terrainH = terrainHeight(xpos, zpos);
    gearDown = (ypos <= terrainH + GROUND_THRESHOLD || ypos <= 0.25f || speed < GROUND_SPEED) ? 1 : 0;

    if (isThrusting && !gearDown)
    {
        speed += THRUST;
        if (speed > MAX_SPEED) speed = MAX_SPEED;
    }

    if (isCharging)
    {
        initParticles(boosters, PARTICLE_COUNT, xpos - 3.0f, ypos - 0.2f, zpos, 0, 0);
    }
    moveParticles(boosters, PARTICLE_COUNT, 0.016f, 0);

    if (speed > 0.0f || isThrusting)
    {
        float leftX, leftY, leftZ, rightX, rightY, rightZ;
        localToWorld(-0.3f, -0.15f, 1.5f, &leftX, &leftY, &leftZ);
        localToWorld(-0.3f, -0.15f, -1.5f, &rightX, &rightY, &rightZ);
        initParticles(afterburner, AFTERBURNER_COUNT / 2, leftX, leftY, leftZ, 0, 0);
        initParticles(&afterburner[AFTERBURNER_COUNT / 2], AFTERBURNER_COUNT / 2, rightX, rightY, rightZ, 0, 0);
    }
    moveParticles(afterburner, AFTERBURNER_COUNT, 0.016f, 0);

    if (speed > 0.0f)
    {
        float wingLeftX, wingLeftY, wingLeftZ, wingRightX, wingRightY, wingRightZ;
        localToWorld(0.4f, 0.0f, 3.0f, &wingLeftX, &wingLeftY, &wingLeftZ);
        localToWorld(0.4f, 0.0f, -3.0f, &wingRightX, &wingRightY, &wingRightZ);
        initParticles(wingTrails, WING_TRAIL_COUNT / 2, wingLeftX, wingLeftY, wingLeftZ, 0, 1);
        initParticles(&wingTrails[WING_TRAIL_COUNT / 2], WING_TRAIL_COUNT / 2, wingRightX, wingRightY, wingRightZ, 0, 1);
    }
    moveParticles(wingTrails, WING_TRAIL_COUNT, 0.016f, 0);

    moveParticles(explosion, PARTICLE_COUNT, 0.016f, 1);

    moveClouds();

    Radar();

    if (speed > 0.0f && gearDown)
    {
        float drag = DECELERATION + 0.0001f * Abs(angleX);
        speed -= drag;
        if (speed < 0.0f) speed = 0.0f;
    }

    if (!gearDown && speed < 0.5f * MAX_SPEED)
    {
        speed = 0.5f * MAX_SPEED;
    }

    if (speed > 0.0f)
    {
        xDelta = speed * cos(radians(direction));
        zDelta = speed * sin(radians(direction));
        yDelta = speed * sin(radians(angleX));
        xpos += xDelta;
        zpos -= zDelta;
        ypos += yDelta;

        if (ypos > MAX_ALTITUDE) ypos = MAX_ALTITUDE;

        if (xpos > WORLD_SIZE) xpos -= 2 * WORLD_SIZE;
        if (xpos < -WORLD_SIZE) xpos += 2 * WORLD_SIZE;
        if (zpos > WORLD_SIZE) zpos -= 2 * WORLD_SIZE;
        if (zpos < -WORLD_SIZE) zpos += 2 * WORLD_SIZE;

        terrainH = terrainHeight(xpos, zpos);
        if (ypos < terrainH)
        {
            printf("Va cham! No va dat lai.\n");
            initParticles(explosion, PARTICLE_COUNT, xpos, ypos, zpos, 1, 0);
            resetSimulation();
        }
    }

    static int frameCount = 0;
    if (frameCount++ % 60 == 0)
    {
        printf(" Toc do: %.2f, Nang luong: %.2f\n", ypos / 1000.0f, speed, charge);
    }
}

// Ve giao dien nguoi dung
void drawHUD()
{
    glMatrixMode(GL_PROJECTION);
    glPushMatrix();
    glLoadIdentity();
    gluOrtho2D(0, 1200, 0, 600);
    glMatrixMode(GL_MODELVIEW);
    glPushMatrix();
    glLoadIdentity();

    glColor3f(0.0f, 1.0f, 0.0f);
    float distance = sqrt(xpos * xpos + zpos * zpos);
    float heading = fmod(direction + 360.0f, 360.0f);
    glRasterPos2f(10, 580);
    char hud[100];
    snprintf(hud, sizeof(hud), "Huong: %.1f deg  Khoang cach: %.1f km", heading, distance);
    for (char* c = hud; *c != '\0'; c++)
    {
        glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
    }

   
    glColor3f(1.0f, 1.0f, 0.0f);
    const char* controls[] = {
        "Phim Space: Nap nang luong (giu space it nhat 2s)",
        "Phim Up: Tang do cao",
        "Phim Down: Giam do cao",
        "Phim Left: Quay trai",
        "Phim Right: Quay phai",
        "Phim p: reset",
        "Phim w: Bat/tat che do khung",
        "Phim c: Doi goc nhin (buong lai)",
        "Phim Esc: Thoat"
    };
    float y_pos = 580.0f;
    for (int i = 0; i < sizeof(controls) / sizeof(controls[0]); i++)
    {
        glRasterPos2f(900, y_pos);
        for (const char* c = controls[i]; *c != '\0'; c++)
        {
            glutBitmapCharacter(GLUT_BITMAP_HELVETICA_12, *c);
        }
        y_pos -= 20.0f;
    }

    drawCompass();
    drawRadar();

    glPopMatrix();
    glMatrixMode(GL_PROJECTION);
    glPopMatrix();
    glMatrixMode(GL_MODELVIEW);
}

// Ham ve chinh
void display()
{
    // Ve canh, may bay, HUD
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    glMatrixMode(GL_MODELVIEW);
    glLoadIdentity();

    if (isCockpitView)
    {
        camx = xpos + 1.7f * cos(radians(direction));
        camy = ypos + 0.1f;
        camz = zpos - 1.7f * sin(radians(direction));
        gluLookAt(camx, camy, camz,
                  camx + cos(radians(direction)), camy + sin(radians(angleX)), camz - sin(radians(direction)),
                  0.0, 1.0, 0.0);
    }
    else
    {
        camx = xpos - 5.0f * cos(radians(direction));
        camz = zpos + 5.0f * sin(radians(direction));
        camy = 2.0f + ypos;
        gluLookAt(camx, camy, camz, xpos, ypos, zpos, 0.0, 1.0, 0.0);
    }

    glClearColor(0.53f, 0.81f, 0.92f, 1.0f);
    GLfloat light_pos[] = {1.0f, 1.0f, 1.0f, 0.0f};
    glLightfv(GL_LIGHT0, GL_POSITION, light_pos);
    glDisable(GL_FOG);

    GLenum err = glGetError();
    if (err != GL_NO_ERROR)
    {
        printf("Loi OpenGL trong display: %d\n", err);
    }

    landmarks();
    drawTerrain();
    drawParticles(boosters, PARTICLE_COUNT, 1, 0);
    drawParticles(afterburner, AFTERBURNER_COUNT, 1, 0);
    drawParticles(wingTrails, WING_TRAIL_COUNT, 1, 1);
    drawParticles(explosion, PARTICLE_COUNT, 1, 0);
    drawClouds();

    glPushMatrix();
    float vibe = speed * 0.05f * sin(glutGet(GLUT_ELAPSED_TIME) * 0.01f);
    glTranslatef(xpos + vibe, ypos, zpos);
    glRotatef(direction, 0.0f, 1.0f, 0.0f);
    glRotatef(angleX, 1.0f, 0.0f, 0.0f);
    glRotatef(angleY, 0.0f, 0.0f, 1.0f);
    plane();
    glPopMatrix();

    drawHUD();
    drawAltitudeBar();

    glutSwapBuffers();
}

// Xu ly phim
void keyboard(unsigned char key, int x, int y)
{
    switch (key)
    {
        case 27:
            exit(0);
            break;
        case ' ':
            if (speed == 0.0f && gearDown)
            {
                isCharging = 1;
                charge += 0.01f;
                if (charge > MAX_CHARGE) charge = MAX_CHARGE;
                printf("Dang nap: %.2f\n", charge);
            }
            break;
        case 'p':
            resetSimulation();
            break;
        case 'w':
            isWireframe = !isWireframe;
            printf("Khung: %s\n", isWireframe ? "ON" : "OFF");
            break;
        case 'c':
            isCockpitView = !isCockpitView;
            printf("Goc nhin: %s\n", isCockpitView ? "Buong lai" : "Ngoai");
            break;
    }
    glutPostRedisplay();
}

// Xu ly tha phim
void keyboardUp(unsigned char key, int x, int y)
{
    switch (key)
    {
        case ' ':
            if (isCharging)
            {
                isCharging = 0;
                speed = charge * MAX_SPEED;
                angleX = charge * 30.0f;
                charge = 0.0f;
                isAutoFlying = 1;
                printf("Khoi dong voi toc do=%.2f, goc nghieng=%.2f\n", speed, angleX);
                printf("Tu dong: ON\n");
            }
            break;
    }
    glutPostRedisplay();
}

// Xu ly phim dac biet
void specialKeys(int key, int x, int y)
{
    switch (key)
    {
        case GLUT_KEY_UP:
            angleX += PITCH_RATE;
            if (angleX > 45.0f) angleX = 45.0f;
            printf("Tang goc nghieng: goc=%.1f\n", angleX);
            break;
        case GLUT_KEY_DOWN:
            angleX -= PITCH_RATE;
            if (angleX < -45.0f) angleX = -45.0f;
            printf("Giam goc nghieng: goc=%.1f\n", angleX);
            break;
        case GLUT_KEY_LEFT:
            direction += YAW_RATE;
            break;
        case GLUT_KEY_RIGHT:
            direction -= YAW_RATE;
            break;
    }
    glutPostRedisplay();
}

// Cap nhat moi khung hinh
void idle()
{
    advanceScene();
    glutPostRedisplay();
}

// Thay doi kich thuoc cua so
void reshape(int width, int height)
{
    glViewport(0, 0, (GLsizei)width, (GLsizei)height);
    glMatrixMode(GL_PROJECTION);
    glLoadIdentity();
    gluPerspective(60.0, (GLfloat)width / (GLfloat)height, 0.1, 200.0);
    glMatrixMode(GL_MODELVIEW);
}

// Khoi tao OpenGL
void initOpenGL()
{
    // Thiet lap anh sang, chat lieu, va cac che do
    GLfloat mat_specular[] = {1.0f, 1.0f, 1.0f, 1.0f};
    GLfloat mat_shininess[] = {100.0f};
    GLfloat light_directional[] = {1.0f, 1.0f, 1.0f, 0.0f};
    GLfloat light_diffuse[] = {1.0f, 1.0f, 1.0f, 1.0f};

    glClearColor(0.53f, 0.81f, 0.92f, 1.0f);
    glShadeModel(GL_SMOOTH);
    glLightfv(GL_LIGHT0, GL_POSITION, light_directional);
    glLightfv(GL_LIGHT0, GL_DIFFUSE, light_diffuse);
    glMaterialfv(GL_FRONT, GL_SHININESS, mat_shininess);
    glMaterialfv(GL_FRONT, GL_SPECULAR, mat_specular);
    glColorMaterial(GL_FRONT, GL_DIFFUSE);

    glEnable(GL_LIGHTING);
    glEnable(GL_LIGHT0);
    glEnable(GL_COLOR_MATERIAL);
    glEnable(GL_DEPTH_TEST);
    glEnable(GL_NORMALIZE);
    glEnable(GL_CULL_FACE);

    glFogi(GL_FOG_MODE, GL_EXP2);
    glFogf(GL_FOG_DENSITY, 0.02f);
    GLfloat fogColor[] = {0.53f, 0.81f, 0.92f, 1.0f};
    glFogfv(GL_FOG_COLOR, fogColor);
    glDisable(GL_FOG);

    GLenum err = glGetError();
    if (err != GL_NO_ERROR)
    {
        printf("Loi OpenGL trong initOpenGL: %d\n", err);
    }

    initParticles(boosters, PARTICLE_COUNT, 0.0f, 0.0f, 0.0f, 0, 0);
    initParticles(afterburner, AFTERBURNER_COUNT, 0.0f, 0.0f, 0.0f, 0, 0);
    initParticles(wingTrails, WING_TRAIL_COUNT, 0.0f, 0.0f, 0.0f, 0, 1);
    initParticles(explosion, PARTICLE_COUNT, 0.0f, 0.0f, 0.0f, 1, 0);
    initClouds();
    initTrees();
    initEnemies();
}

// Ham chinh
int main(int argc, char* argv[])
{
    // Khoi tao GLUT va chay vong lap chinh
    srand((unsigned int)time(NULL));
    if (time(NULL) == -1)
    {
        printf("Loi: Khong lay duoc thoi gian cho random.\n");
        return 1;
    }

    glutInit(&argc, argv);
    glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB | GLUT_DEPTH);
    glutInitWindowSize(1200, 600);
    glutInitWindowPosition(100, 100);
    glutCreateWindow("Mo phong may bay voi tu dong");

    initOpenGL();

    glutDisplayFunc(display);
    glutReshapeFunc(reshape);
    glutKeyboardFunc(keyboard);
    glutKeyboardUpFunc(keyboardUp);
    glutSpecialFunc(specialKeys);
    glutIdleFunc(idle);

    glutMainLoop();
    
    system("PAUSE");
    return 0;
}
