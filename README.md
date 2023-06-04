# MFS
Minimalist read-only file system for microcontroller

Read the full story at [AdAstra-Soft.com](https://adastra-soft.com/a-minimalist-file-system/)

I needed a file system for a small web server managed by an STM32G071 microcontroller.
For this server on microcontroller I need to access web pages which are read-only files. Indeed the pages are created on a PC and gathered in a folder, this folder is then treated to constitute only one file containing the file system to be downloaded in the flash managed by the microcontroller.

After some research I decided to write my own minimalist file system to meet my needs: the MFS library.

## How to use the MFS
- Build the `mfsBuid` and `mfsExtract` utilities on your favourite Windows or Linux (non tested) PC.
- Gather the files for your file system in a folder, you can use sub folders.
- Use the `mfsBuid` utility to build the file containing the MFS
syntax:
- The mfsExtract utility makes it possible to extract the contents of an MFS and to reconstruct its tree structure on the PC. Then it is possible to compare the original tree structure used to build the MFS and the tree structure extracted from the MFS. If all went well, the directories and files are identical.


## The MFS library API
```
mfsError_t      mfsMount    (mfsCtx_t * pCtx) ;
mfsError_t      mfsUmount   (mfsCtx_t * pCtx) ;
mfsError_t      mfsStat     (mfsCtx_t * pCtx, const char * path, mfsStat_t * pStat) ;

mfsError_t      mfsOpen     (mfsCtx_t * pCtx, mfsFile_t * pFile, const char * path) ;
mfsError_t      mfsClose    (mfsFile_t * pFile) ;

mfsError_t      mfsRead     (mfsFile_t * pFile, void * pBuffer, int32_t size) ;
mfsError_t      mfsSeek     (mfsFile_t * pFile, int32_t offset, mfsWhenceFlag_t whence) ;
int32_t         mfsSize     (mfsFile_t * pFile) ;
void            mfsRewind   (mfsFile_t * pFile) ;

mfsError_t      mfsDirOpen  (mfsCtx_t * pCtx, const char * path, mfsDir_t * pDir) ;
mfsError_t      mfsDirRead  (mfsCtx_t * pCtx, mfsDir_t * pDir, mfsDirEntry_t * pEntry) ;
```

## Code example
This example runs on AdAstra RTOS and is easily portable to Windows or Linux for testing.
Have a look to the `mfsBuild.c` and `mfsExtact.c`.
```
/*
----------------------------------------------------------------------
	
	Alain Chebrou

	mfsTest.c	Some examples using MFS

	When		Who	What
	05/31/23	ac	Creation

----------------------------------------------------------------------
*/

#include	"aa.h"
#include	"mfs.h"

#define		printf		aaPrintf

//--------------------------------------------------------------------------------

#include	"w25q.h"		// Flash

// The user provided function to set in the MFS context

static	int mfsDevRead (void * userData, uint32_t address, void * pBuffer, uint32_t size)
{
	W25Q_Read (pBuffer, address + (uint32_t) userData, size) ;
	return MFS_ENONE ;
}

static	void mfsDevLock (void * userData)
{
	(void) userData ;
	W25Q_SpiGet () ;
}

static	void mfsDevUnlock (void * userData)
{
	(void) userData ;
}

//--------------------------------------------------------------------------------
// Example of reading an MFS file

void	dumpFile (const char * filePath)
{
	mfsCtx_t	mfsCtx ;
	mfsFile_t	mfsFile ;
	mfsStat_t	fileStat ;
	mfsError_t	err ;
	uint32_t	nn, fileSize ;
	uint8_t		readBuffer [80] ;

	// Initialize the MFS context
	// userData is used to provide the starting address of the file system in the FLASH
	mfsCtx.userData = (void *) 0x100000 ;
	mfsCtx.lock     = mfsDevLock ;
	mfsCtx.unlock   = mfsDevUnlock ;
	mfsCtx.read     = mfsDevRead ;

	// Mount the MFS
	err = mfsMount (& mfsCtx) ;
	if (err != MFS_ENONE)
	{
		printf ("mfsMount error: %d\nAbort\n", err) ;
		return ;
	}

	// Test if the file exist
	err = mfsStat (& mfsCtx, filePath, & fileStat) ;
	if (err != MFS_ENONE)
	{
		printf ("mfsStat error: %d\nAbort\n", err) ;
		return ;
	}
	printf ("Information from: mfsStat (\"%s\")\n", filePath) ;
	printf ("  Type %s\n", ((fileStat.type & MFS_DIR) != 0) ? "DIR" : ((fileStat.type & MFS_FILE) != 0) ? "FILE" : "???") ;
	printf ("  Size %d\n\n", fileStat.size) ;

	if ((fileStat.type & MFS_DIR) == MFS_DIR)
	{
		printf ("Can't dump a directory\n") ;
		return ;
	}

	// Open the MFS file
	err = mfsOpen (& mfsCtx, & mfsFile, filePath) ;
	if (err != MFS_ENONE)
	{
		printf ("msfOpen error: %d\nAbort\n", err) ;
		return ;
	}

	// Dump the file
	printf ("File content:\n") ;
	fileSize = 0 ;
	printf ("<") ;	// Start marker
	while (0 != (nn = mfsRead (& mfsFile, readBuffer, sizeof (readBuffer))))
	{
		for (uint32_t ii = 0 ; ii < nn ; ii++)
		{
			aaPutChar (readBuffer [ii]) ;
		}
		fileSize += nn;
	}
	printf (">\nBytes read: %d\n", fileSize) ;	// End marker

	mfsClose  (& mfsFile) ;
	mfsUmount (& mfsCtx) ;
}

//--------------------------------------------------------------------------------
//--------------------------------------------------------------------------------

//	The files to dump:
const char	* filePath [] =
{
	"/fonts/fonts2",
	"/js/npm.js",
	"/css/ie10-viewport-bug-workaround.css",
} ;

const	uint32_t	filePathCount = sizeof (filePath) / sizeof (char *) ;

//--------------------------------------------------------------------------------

void	mfsTest (void)
{
	for (uint32_t ii = 0 ; ii < filePathCount ; ii++)
	{
		dumpFile (filePath [ii]) ;
		printf ("\n-------------------------------\n") ;
	}
}

//--------------------------------------------------------------------------------
```
